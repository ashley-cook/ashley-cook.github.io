---
title: "Building a Replay System for Town of Salem"
date: 2024-06-14T13:12:57-07:00
draft: false
---

[Town of Salem](https://www.blankmediagames.com/) is an online multiplayer social deduction video game based on
the [Mafia](https://en.wikipedia.org/wiki/Mafia_(party_game)) party game. The game is characterized by rapid
rounds of quick judgements and information asymmetries. Confusion is a resource, strategy is a sword, and 
expectation is a weakness. Being able to review previous games and study how each choice you made determined your opportunities and 
outcomes would be of great advantage in refining game skill.

## Assessment
The Steam distribution of the game is written in the [Unity](https://unity.com/) game engine and compiled into .NET assemblies 
which are loaded and executed by the engine at runtime. This means that we're
able to decompile the game code with nearly one-to-one accuracy. Owing to the 
high success rate of lossless transpiling of Intermediate Language bytecode, 
many games written in Unity opt to obfuscate their release assemblies, 
especially since many development teams choose to externalize costs at the
expense of the players by making cheating both possible and easy. 

Upon reviewing the primary assembly for Town of  Salem, no obfuscation is employed. The networking code is engineered 
to prevent players from exploiting the game with client-side modifications. This is valuable in our case, as it 
means that we'll be able to record network traffic during live games and simply feed everything back in later. 
Effectively no interaction on the client is necessary to progress the state of the game. Additionally, the lack
of obfuscation will make the general task of writing a mod much easier.

## Technical Approach
I've opted to use an existing modding framework called [BepInEx](https://github.com/BepInEx/BepInEx). It utilizes
DLL hijacking to perform the necessary redirections, which is important as it means we'll almost never need to
update the mod or re-patch existing files after a game update. It also provides a 
[custom patching library](https://github.com/BepInEx/HarmonyX), which will allow me to easily patch the network
management code. I'll also be working from the GUI system referenced in my [previous post](posts/imgui-in-unity/).

## Network Overview
The networking layer is directed through a small class `TOSNetworkService`. This class is responsible for managing
the lifetime of the TCP socket, testing the health of the connection, and serializing/deserializing network traffic.
The latter task is the most relevant, since I'll want to both save raw game traffic to disk and prevent the 
client from attempting to send traffic during replay playback.

```cs
public class TOSNetworkService : INetworkService 
{
    // ...
    
    public void ProcessMessages(byte[] data) {}
    
    public void SendMessage(Client.BaseMessage msg) {}
    
    // ...
}
```

Since the networking protocol is so straightforward, the reverse-engineering part of the project is over 
almost instantly.

## Building the Perfect Beast
I only need a few points of interface for replay management, so I'll start by defining those externally-accessible
methods immediately.
```cs
static class Replay
{
    public static bool IsRecording { get; private set; }
    public static bool IsPlaybackActive { get; private set; }
    public static bool IsPlaybackPaused { get; private set; }

    public static void OnServerPacket(byte[] data);
    
    public static void StartPlayback(String filePath);
    
    public static void StopPlayback();
    
    public static void TogglePlayback();
}
```

The requisite patches are very simple:
```cs
[HarmonyPatch(typeof(TOSNetworkService))]
class PatchTOSNetworkService
{
    [HarmonyPatch(nameof(TOSNetworkService.ProcessMessages))]
    [HarmonyPatch(new Type[] { typeof(byte[]) })]
    [HarmonyPrefix]
    public static void PrefixProcessMessages(ref byte[] data)
    {
        Replay.OnServerPacket(data);
    }
    
    [HarmonyPatch(nameof(TOSNetworkService.SendMessage))]
    [HarmonyPatch(new Type[] { typeof(BaseMessage) })]
    [HarmonyPrefix]
    public static bool PrefixSendMessage(ref BaseMessage msg)
    {
        // suppress outbound network traffic when replay files are loaded
        return !Replay.IsPlaybackActive;
    }
}
```

### Recording Format
For recording, I need to capture an array of raw network messages with the time in milliseconds after the start
of the recording that they were received. The playback mechanism is simple: start a timer and check the packet
queue each game tick to determine if the next packet should be processed according to its stored delay.
Fine-grained playback control can be further built on top of that core system later.
```cs
// ... in Replay.cs

private struct ReceivedPacket
{
    public long millisecondDelay;
    public String payloadBase64;
}

private struct Recording
{
    public long startTimeUnixMilli;
    public long lengthInMilliseconds;
    public List<ReceivedPacket> packets;
}

private static DateTime startTime;
private static Stopwatch stopwatch = new Stopwatch();
private static List<ReceivedPacket> packetLog;
```

I want to automatically begin recording each time a game starts and stop recording when it ends. That's simple:
I can watch the packet stream to capture these events.
```cs
// ... in Replay.cs

public static void OnServerPacket(byte[] data)
{
    if (IsPlaybackActive)
    {
        return;
    }

    MessageType messageType = (MessageType) data[0];
    if (messageType == MessageType.CreateLobby)
    {
        StartRecording();
    }

    if (IsRecording)
    {
        packetLog.Add(new ReceivedPacket()
        {
            millisecondDelay = stopwatch.ElapsedMilliseconds,
            payloadBase64 = Convert.ToBase64String(data, Base64FormattingOptions.None)
        });
    }

    if (messageType == MessageType.ReturnToHome)
    {
        StopRecording();
    }
}
```

Wonderful! The simplicity of the network protocol implementation means that I don't have to worry anything like
frame delimitation, which can get very complicated depending on the composition of the protocol.

### Saving and Loading
Next up, serializing the recording and writing to disk. I could come up with a low-level format to store
recordings in a space-efficient manner, but it won't be a cost-effective use of my time. The network protocol
is already rather sparse, since there's nothing to communicate except the bare gameplay fundamentals. It would also
make further changes to the recording format somewhat frustrating. By using JSON, I'm able to use existing
libraries to serialize and deserialize C# objects in a flash while ignoring object fields that are not present
in a given recording file. Simple code is more flexible and less prone to failure!
```cs
// ... in Replay.cs

private static void StopRecording()
{
    if (!IsRecording)
    {
        return;
    }

    // determine file name
    TimeSpan epochTimeSpan = startTime.ToUniversalTime() - new DateTime(1970, 1, 1);
    long epochMillis = (long) epochTimeSpan.TotalMilliseconds;
    var fileName = String.Format("replay/{0}.json", epochMillis);

    // populate the recording struct
    Recording recording = new Recording();
    recording.startTimeUnixMilli = epochMillis;
    recording.lengthInMilliseconds = (long) DateTime.Now.Subtract(startTime).TotalMilliseconds;
    recording.packets = packetLog;

    // serialize the recording and write to disk
    Directory.CreateDirectory("replay");
    string json = Newtonsoft.Json.JsonConvert.SerializeObject(recording);
    File.WriteAllText(fileName, json);

    IsRecording = false;
}

private static void StartRecording(String filePath)
{
    Recording recording = Newtonsoft.Json.JsonConvert.DeserializeObject<Recording>(File.ReadAllText(filePath));
    
    // ...
}
```

### Emulating Realtime Playback
Earlier, I said the phrase "packet queue" as a means of conceptualizing the received server traffic, so I'll create a `Queue<ReceivedPacket>` to store packets from the
loaded recording file. Since each packet is stored with the millisecond delay it was received after recording, 
and since each packet was serialized in the order it was received, emulating linear playback is simply a matter
of checking the packet at the front of the queue and comparing it to the playback timer.
```cs
// ... in Replay.cs

private static Queue<ReceivedPacket> replayablePackets = new Queue<ReceivedPacket>();

public static void StartPlayback(String filePath)
{
    Recording recording = Newtonsoft.Json.JsonConvert.DeserializeObject<Recording>(File.ReadAllText(filePath));

    replayablePackets.Clear();
    foreach (ReceivedPacket packet in recording.packets)
    {
        replayablePackets.Enqueue(packet);
    }
    
    stopwatch.Restart();
    playbackOffset = 0;
    IsPlaybackActive = true;
    IsPlaybackPaused = false;
}

private static void TickPlayback() 
{
    while (replayablePackets.Count > 0 
        && replayablePackets.Peek().millisecondDelay <= stopwatch.ElapsedMilliseconds
    {
        ReceivedPacket packet = replayablePackets.Dequeue();
        byte[] data = Convert.FromBase64String(packet.payloadBase64);
        TOSNetworkService networkService = (TOSNetworkService) GlobalServiceLocator.Instance.NetworkService;
        networkService.ProcessMessages(data);
    }
}
```

I also mentioned checking the packet queue each game tick, so I'll need to utilize my `BepInLoader` class
(which extends Unity's `MonoBehavior` class, and is thus integrated into the engine's ECS implementation) to
execute the playback logic in the main thread.

```cs
public class BepInLoader : BaseUnityPlugin
{
    // ...
    
    public static event Action OnUpdate;
    private void Update()
    {
        if (OnUpdate != null) 
        {    
            OnUpdate();
        }
    }
}

static class Replay 
{
    // ...
    
    public static void StartPlayback() 
    {
        // ...
        
        BepInLoader.OnUpdate += TickPlayback;
    }
    
    public static void StopPlayback()
    {
        // ...
        
        BepInLoader.OnUpdate -= TickPlayback;
    }
}
```

### Further Tweaks

With all of this done, simple playback works better than I had expected. I was able to toggle the playback state
by simply stopping and starting the `Stopwatch` object, but sometimes there are long pauses that I'd rather skip.
I need to come up with a way to skip the timer ahead so that I can force the next packet to be processed immediately.

Since the `Stopwatch` class doesn't expose any methods to modify the time, I'll have to track the time offset
independently.

```cs
// ... in Replay.cs

private static Queue<ReceivedPacket> replayablePackets = new Queue<ReceivedPacket>();
private static long playbackOffset = 0;

public static void SkipPacket()
{
    if (PlaybackActive && replayablePackets.Count > 0)
    {
        ReceivedPacket packet = replayablePackets.Peek();
        playbackOffset = packet.millisecondDelay - stopwatch.ElapsedMilliseconds;
    }
}

private static void TickPlayback()
{
    while (replayablePackets.Count > 0 
        && replayablePackets.Peek().millisecondDelay <= (stopwatch.ElapsedMilliseconds + playbackOffset))
    {
        // ...
    }
}
```

Initially, I ran into a bug with this approach where the playback seemed to jump ahead much further than expected.
It wasn't quite until I realized that the problem worsened during later playback time that I realized my issue: 
I had forgotten to compensate for the amount of time that had already elapsed. Simply subtracting 
`stopwatch.ElapsedMilliseconds` from `packet.millisecondDelay` corrected the issue.

### Final Improvement

Now that recording and playback are working without issue, I'm able to review previous games and study my mistakes.
Yet, there's one final feature that I'm really missing: seeing which role players were assigned! As long as I 
stick around long enough, the server displays a post-game summary that reveals all the role assignments. I can
access this at the beginning of playback by peeking through the packet queue and checking the message opcode 
until I discover the correct packet:

```cs
// ... in Replay.cs

public static AfterGameScreenDataMessage PostGameSummary { get; private set; }

public static AfterGameScreenDataMessage GetPostGameSummary()
{
    ReceivedPacket postGamePacket = replayablePackets.Where((pkt) =>
    {
        byte[] data = Convert.FromBase64String(pkt.payloadBase64);
        MessageType type = (MessageType) data[0];
        return type == MessageType.AfterGameScreenData;
    }).FirstOrDefault();
    
    if (postGamePacket.Equals(null))
    {
        return null;
    }

    byte[] data = Convert.FromBase64String(packet.payloadBase64);
    return new AfterGameScreenDataMessage(data);
}

public static void StartPlayback(String filePath)
{
    // ...
    
    PostGameSummary = GetPostGameSummary();
}
```

Rendering this information with Dear ImGui is a piece of cake:
```cs
// ... in GUI/Overlays/ReplayPostGameOverlay.cs

public static void Render()
{
    if (Replay.IsPlaybackActive && Replay.PostGameSummary == null)
    {
        return;
    }

    ImGui.SetNextWindowBgAlpha(0.75f);

    ImGuiWindowFlags windowFlags = ImGuiWindowFlags.None;
    windowFlags |= ImGuiWindowFlags.AlwaysAutoResize;
    windowFlags |= ImGuiWindowFlags.NoFocusOnAppearing;
    windowFlags |= ImGuiWindowFlags.NoNav;

    if (ImGui.Begin("Replay: Post-game", windowFlags))
    {
        ImGuiTableFlags tableFlags = ImGuiTableFlags.None;
        tableFlags |= ImGuiTableFlags.Borders;
        tableFlags |= ImGuiTableFlags.Resizable;
        if (ImGui.BeginTable("table_postgame_players", 4, tableFlags))
        {
            ImGui.TableSetupColumn("Position", ImGuiTableColumnFlags.DefaultSort);
            ImGui.TableSetupColumn("Nick");
            ImGui.TableSetupColumn("Account");
            ImGui.TableSetupColumn("Roles");

            ImGui.TableHeadersRow();

            GameRules gameRules = GlobalServiceLocator.GameRulesService.GetGameRules();

            foreach (EndGamePartyMemberInfo info in Replay.PostGameSummary.PlayerList.OrderBy(x => x.Position))
            {
                ImGui.TableNextRow();

                ImGui.TableSetColumnIndex(0);
                ImGui.Text("" + (info.Position + 1));
                
                ImGui.TableSetColumnIndex(1);
                ImGui.Text(info.OriginalName);
                
                ImGui.TableSetColumnIndex(2);
                ImGui.Text(info.AccountName);
                
                ImGui.TableSetColumnIndex(3);
                Role role = null;
                gameRules.Roles.TryGetValue(info.BeginningRoleId, out role);
                ImGui.TextColored(role.Color.ToVec4(), role.Name);
            }

            ImGui.EndTable();
        }
    }
}
```

Works like a charm!

![ImGui window rendering the post-game summary of the active replay](/images/posts/town-of-salem-replay/summary_overlay.png)

## Future Changes
The last remaining problem with this approach is that it lacks some client-side state replication. Since the 
recording only stores inbound packets from the server, all stateful UI interaction (typing messages, writing notes, 
selecting targets) is lost.

In the future, I plan to address this by tracking each of these in more detail
and inserting them as meta-packets inside the recording that require special handling by the playback emulator.
However, I'm very pleased with the current result, so those changes will have to wait for another day.
