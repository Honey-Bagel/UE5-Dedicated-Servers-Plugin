# Unreal Engine 5 Multiplayer Dedicated Server

A complete multiplayer game server implementation using Unreal Engine 5 (C++) with AWS GameLift integration for scalable, production-ready game hosting and matchmaking.

**Certificate Completion:** [Unreal Engine 5 Dedicated Servers with AWS and GameLift](https://www.udemy.com/certificate/UC-85dc6c28-c6df-44af-a525-cc5def5fc89f/)

![Unreal Engine](https://img.shields.io/badge/Unreal%20Engine-5-313131?style=for-the-badge&logo=unrealengine&logoColor=white)
![C++](https://img.shields.io/badge/C++-00599C?style=for-the-badge&logo=c%2B%2B&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white)
![GameLift](https://img.shields.io/badge/AWS%20GameLift-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)

---

## Features

### Server Architecture
- **Dedicated Server Implementation** - Server-authoritative gameplay preventing client-side cheating
- **Client-Server Replication** - Property replication and RPCs for multiplayer state synchronization
- **Seamless Travel** - Smooth transitions between lobby and match maps without disconnecting players
- **Player Session Management** - AWS GameLift integration for player authentication and session validation

### AWS Cloud Integration
- **GameLift Fleet Management** - Auto-scaling server instances based on player demand
- **Multi-Region Deployment** - Server infrastructure across multiple AWS regions (EC2, S3)
- **Matchmaking System** - Automated game session creation and player session assignment
- **Health Monitoring** - Automatic health checks and instance replacement for high availability

### Game Systems
- **Lobby System** - Pre-match lobby with real-time player list synchronization using Fast Array Serialization
- **Match Flow** - Complete match lifecycle (Pre-match → Match → Post-match) with countdown timers
- **Player Authentication** - JWT-based authentication with AWS Cognito integration
- **Stats & Leaderboards** - RESTful API integration for persistent player statistics and rankings

### Networking Features
- **Server-Client RPC Communication** - Remote procedure calls for player actions and server responses
- **Network Replication** - Efficient delta serialization for player info and game state
- **Latency Compensation** - Client-side prediction with server reconciliation
- **Connection Management** - Graceful handling of player connections, disconnections, and session cleanup

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Application                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  Portal UI   │  │  Lobby UI    │  │  Match Gameplay      │   │
│  │  (Sign In)   │  │  (Players)   │  │  (Timer, Stats)      │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTP Requests (Auth, Matchmaking)
                             │ UE5 Replication (Game State)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       AWS GameLift                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  GameLift Fleets (Auto-Scaling Server Instances)         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │   │
│  │  │  Lobby      │  │  Match      │  │  Match      │       │   │
│  │  │  Server 1   │  │  Server 1   │  │  Server 2   │ ...   │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────┐  ┌──────────────────────────────────┐      │
│  │  Matchmaking    │  │  Player Session Management       │      │
│  │  Queue          │  │  (AcceptPlayerSession)           │      │
│  └─────────────────┘  └──────────────────────────────────┘      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ AWS SDK Calls
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Backend Services (AWS)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐    │
│  │  Cognito     │  │  API Gateway │  │  Lambda Functions   │    │
│  │  (Auth)      │  │  (REST APIs) │  │  (Game Stats)       │    │
│  └──────────────┘  └──────────────┘  └─────────────────────┘    │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  DynamoDB (Player Stats & Leaderboard Storage)           │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Technical Stack

### Game Engine & Language
- **Unreal Engine 5** (C++)
- **Blueprints** (UI and visual scripting)

### Networking & Multiplayer
- **UE5 Replication System** - Property replication with `DOREPLIFETIME` macros
- **Fast Array Serialization** - Efficient delta serialization for dynamic arrays
- **Server RPC** - `Server_Reliable` for client-to-server communication
- **Client RPC** - `Client_Reliable` for server-to-client updates

### AWS Services
- **GameLift** - Dedicated server fleet management, matchmaking, and auto-scaling
- **EC2** - Compute instances for game servers
- **S3** - Server build storage and versioning
- **Cognito** - User authentication and JWT token management
- **API Gateway** - RESTful API endpoints
- **Lambda** - Serverless functions for game stats processing
- **DynamoDB** - NoSQL database for player data and leaderboards

### Key Dependencies
- **GameLift Server SDK** - Integration with AWS GameLift services
- **HTTP Module** - RESTful API communication
- **JSON** - Data serialization for API requests/responses
- **UMG (Unreal Motion Graphics)** - UI system
- **Gameplay Tags** - Organized API endpoint management

---

## Key Implementation Details

### 1. **GameLift Server Initialization**
```cpp
// DS_LobbyGameMode.cpp - Lines 94-138
void ADS_LobbyGameMode::InitGameLift()
{
    FServerParameters ServerParameters;
    SetServerParameters(ServerParameters); // Parse command-line args
    DSGameInstanceSubsystem->InitGameLift(ServerParameters);
}
```
- Parses command-line parameters (`authtoken`, `hostid`, `fleetid`, `websocketurl`)
- Registers `OnStartGameSession`, `OnTerminate`, and `OnHealthCheck` callbacks
- Calls `ProcessReady()` to notify GameLift the server is ready for players

### 2. **Player Session Validation**
```cpp
// DS_LobbyGameMode.cpp - Lines 58-90
void ADS_LobbyGameMode::TryAcceptPlayerSession(...)
{
    // Describe player session from GameLift
    auto DescribePlayerSessionsOutcome = 
        Aws::GameLift::Server::DescribePlayerSessions(Request);
    
    // Verify session status is RESERVED
    if (PlayerSession.GetStatus() == PlayerSessionStatus::RESERVED)
    {
        Aws::GameLift::Server::AcceptPlayerSession(PlayerSessionId);
    }
}
```
- Validates player sessions with GameLift before allowing login
- Prevents unauthorized connections and session hijacking

### 3. **Network Replication with Fast Arrays**
```cpp
// LobbyPlayerInfo.h - Lines 21-35
USTRUCT()
struct FLobbyPlayerInfoArray : public FFastArraySerializer
{
    GENERATED_BODY()
    
    TArray<FLobbyPlayerInfo> Players;
    
    bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParams)
    {
        return FastArrayDeltaSerialize<FLobbyPlayerInfo, 
            FLobbyPlayerInfoArray>(Players, DeltaParams, *this);
    }
    
    void AddPlayer(const FLobbyPlayerInfo& NewPlayerInfo);
    void RemovePlayer(const FString& Username);
};
```
- Efficiently replicates only changed elements (delta serialization)
- Minimizes network bandwidth for dynamic player lists

### 4. **Client-Server Latency Compensation**
```cpp
// DSPlayerController.cpp - Lines 44-55
void ADSPlayerController::Server_Ping_Implementation(float TimeOfRequest)
{
    Client_Pong(TimeOfRequest); // Echo back to client
}

void ADSPlayerController::Client_Pong_Implementation(float TimeOfRequest)
{
    const float RoundTripTime = GetWorld()->GetTimeSeconds() - TimeOfRequest;
    SingleTripTime = RoundTripTime * 0.5f; // One-way latency
}
```
- Calculates single-trip network latency for timer synchronization
- Compensates for network delay in countdown timers

### 5. **Seamless Map Travel**
```cpp
// DS_GameModeBase.cpp - Lines 68-78
void ADS_GameModeBase::TrySeamlessTravel(TSoftObjectPtr<UWorld> DestinationMap)
{
    const FString MapName = DestinationMap.ToSoftObjectPath().GetAssetName();
    if (GIsEditor)
    {
        UGameplayStatics::OpenLevelBySoftObjectPtr(this, DestinationMap);
    }
    else
    {
        GetWorld()->ServerTravel(MapName); // Seamless travel on dedicated server
    }
}
```
- Transitions between lobby and match without player disconnections
- Preserves player state and connections across map changes

### 6. **RESTful API Integration**
```cpp
// GameSessionsManager.cpp - Lines 20-35
void UGameSessionsManager::JoinGameSession()
{
    TSharedRef<IHttpRequest> Request = FHttpModule::Get().CreateRequest();
    Request->SetURL(APIData->GetAPIEndpoint(
        DedicatedServersTags::GameSessionsAPI::FindOrCreateGameSession));
    Request->SetVerb(TEXT("POST"));
    Request->SetHeader(TEXT("Authorization"), AccessToken); // JWT auth
    Request->ProcessRequest();
}
```
- Communicates with AWS API Gateway for matchmaking
- Handles asynchronous HTTP responses with callbacks

### 7. **Match Statistics & Leaderboards**
```cpp
// GameStatsManager.cpp - Lines 91-107
void UGameStatsManager::UpdateLeaderboard(const TArray<FString>& WinnerUsernames)
{
    TSharedPtr<FJsonObject> JsonObject = MakeShareable(new FJsonObject());
    TArray<TSharedPtr<FJsonValue>> PlayerIdJsonArray;
    
    for(const FString& Username : WinnerUsernames)
    {
        PlayerIdJsonArray.Add(MakeShareable(new FJsonValueString(Username)));
    }
    
    JsonObject->SetArrayField(TEXT("playerIds"), PlayerIdJsonArray);
    // POST to Lambda function for DynamoDB update
}
```
- Records match results to DynamoDB via Lambda functions
- Updates leaderboard rankings in real-time

---

## Game Flow

### 1. **Authentication Flow**
```
Player Opens Game → Sign In UI → Cognito Authentication → JWT Tokens Stored
                                                           ↓
                                    Access Token (Auth) | Refresh Token (75 min refresh)
```

### 2. **Matchmaking Flow**
```
Click "Join Game" → API: FindOrCreateGameSession → GameLift Creates/Finds Session
                                                    ↓
                    API: CreatePlayerSession → PlayerSessionId + IP:Port
                                               ↓
                    Client Connects to Dedicated Server → PreLogin Validation
                                                           ↓
                    GameLift.AcceptPlayerSession() → Player Joins Lobby
```

### 3. **Match Lifecycle**
```
Lobby (Wait for Players) → Countdown → Seamless Travel → Match Map
                                                          ↓
        Pre-Match Timer (30s) → Match Timer (5 min) → Post-Match Timer (10s)
                                                       ↓
        Stats Recorded to DynamoDB → Leaderboard Updated → Travel to Lobby
```

---

## Key Gameplay Systems

### Timer System
- **Server-Authoritative** - All timers run on server
- **Client Synchronization** - Latency compensation using ping/pong RPC
- **Dynamic Updates** - Countdown broadcasts every 1 second to all clients
- **Types**: Lobby Countdown, Pre-Match, Match, Post-Match

### Lobby System
- **Fast Array Replication** - Efficient player list synchronization
- **Dynamic UI Updates** - Automatic player join/leave notifications
- **Server Validation** - All player additions/removals validated server-side

### Player Session Management
- **GameLift Integration** - `AcceptPlayerSession` on PreLogin
- **Cleanup on Disconnect** - `RemovePlayerSession` on Logout
- **Secure Connections** - PlayerSessionId validation prevents unauthorized access

---

## Configuration

### Server Launch Parameters
```bash
--authtoken=<AUTH_TOKEN>     # From AWS GameLift GetComputeAuthToken API
--hostid=<COMPUTE_NAME>      # GameLift Anywhere host identifier
--fleetid=<FLEET_ID>         # GameLift fleet ID
--websocketurl=<WS_URL>      # GameLift service SDK endpoint
--port=<PORT>                # Game server port (default: 7777)
```

### API Endpoints (Configured via Data Assets)
- **GameSessions API**: `ListFleets`, `FindOrCreateGameSession`, `CreatePlayerSession`
- **Portal API**: `SignUp`, `ConfirmSignUp`, `SignIn`, `SignOut`
- **GameStats API**: `RecordMatchStats`, `RetrieveMatchStats`, `UpdateLeaderboard`, `RetrieveLeaderboard`

---

## Learning Outcomes

Through this project, I gained hands-on experience with:

✅ **Dedicated Server Architecture** - Building server-authoritative multiplayer games  
✅ **Cloud Infrastructure** - Deploying and managing AWS GameLift fleets  
✅ **Network Replication** - Implementing efficient state synchronization patterns  
✅ **Matchmaking Systems** - Creating automated player session management  
✅ **RESTful API Integration** - Connecting game clients to backend services  
✅ **Scalability** - Designing systems to handle multiple concurrent game sessions  
✅ **Player Authentication** - Securing multiplayer games with JWT tokens  
✅ **Production DevOps** - Health monitoring, auto-scaling, and deployment strategies  

---

## Notes

- All C++ code follows Unreal Engine coding standards with proper `UPROPERTY` and `UFUNCTION` specifiers
- Server logic is separated from client logic using `HasAuthority()` checks
- Network traffic is minimized through delta serialization and efficient RPC usage
- AWS credentials are parsed from command-line arguments (not hardcoded)
- The project demonstrates production-ready patterns for commercial multiplayer games

---

## Resources

- [Unreal Engine Network Replication](https://docs.unrealengine.com/5.0/en-US/networking-overview-for-unreal-engine/)
- [AWS GameLift Documentation](https://docs.aws.amazon.com/gameliftservers/latest/developerguide/gamelift-intro.html)

---

**Certificate Earned:** July 2025  
**Technologies:** Unreal Engine 5, C++, AWS (GameLift, EC2, S3, Cognito, Lambda, DynamoDB)
