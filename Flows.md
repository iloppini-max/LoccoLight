# LoccoLight - App Flows

Questo documento descrive visivamente i flussi implementati nell'app LoccoLight, divisi tra **User Flow (Navigazione)** e **Logic Flow (Logiche di Business e State Management)**.

## 1. User Flow (Navigazione delle Schermate)

Rappresenta il percorso visivo che l'utente può intraprendere navigando tra i file nella cartella `app/`.

```mermaid
graph TD
    %% Entry Point
    Entry[App Initialization \n app/index] --> AuthCheck{Is Logged & Complete?}

    %% Onboarding Flow
    AuthCheck -->|No| OB_Welcome[Onboarding: Welcome]
    OB_Welcome --> OB_Auth[Email/Login]
    OB_Auth --> OB_Profile[Set Profile Details]
    OB_Profile --> OB_Avail[Set Availability]
    OB_Avail --> OB_Trans[Transition Screen]
    OB_Trans --> MainTabs

    %% Main App (Tabs)
    AuthCheck -->|Yes| MainTabs[Main Tabs Layout]
    
    MainTabs --> Tab_Home[Tab: Home \n tabs/index]
    MainTabs --> Tab_RunBuddy[Tab: RunBuddy \n tabs/runbuddy]
    MainTabs --> Tab_Messages[Tab: Messages \n tabs/messages]

    %% Home Branches
    Tab_Home --> Discovery[Discovery / Search]
    Tab_Home --> Create[Create Event]
    Tab_Home --> Community[Community Feed]
    #Tab_Home --> Classes[Coaching Classes]
    #Tab_Home --> Challenges[Challenges]
    Tab_Home --> ProfileNav[Profile Menu]

    %% Deep Links from Home
    Discovery --> RunDetail[Run Detail \n run-detail/id]
    Community --> ClubDetail[Club Detail \n club-detail/id]
    Community --> UserDetail[User Profile \n user-detail]
    Classes --> ClassDetail[Class Detail \n class-detail/id]

    %% Profile Branches
    ProfileNav --> ProfileSettings[Settings]
    ProfileNav --> ProfileEdit[Edit Profile]
    ProfileNav --> Wallet[Wallet & Tokens]
    ProfileNav --> Activities[My Activities]
    
    %% Run Interactions Actions
    RunDetail --> Application[Apply to Run]
    RunDetail --> CheckIn[Event Check-In \n check-in/id]
    RunDetail --> Review[Rate Runners \n review/id]
    RunDetail --> MemberRequests[Organizer: Manage Requests]
```

---

## 2. Logic & System Workflow (State Management)

Rappresenta le logiche interne di sistema gestite dagli Store (es. `userStore`, `applicationStore`, `runBuddyStore`, `walletStore`).

### A. Core Application & Premium Logic (`applicationStore`)

Definisce come vengono processate le iscrizioni agli eventi in base allo status dell'utente.

```mermaid
flowchart TD
    Apply[User requests to join an Event] --> TypeCheck{Is Request 'Standard' \n or 'Instant' (Locco-Match)?}
    
    %% Standard Request
    TypeCheck -->|Standard| PremiumCheck{Is User Premium?}
    PremiumCheck -->|Yes| AutoApprove[Auto-Approved \n success: true]
    PremiumCheck -->|No| FreeCheck{Has Active Apps < 1?}
    FreeCheck -->|No| AppReject[Reject: Max 1 active booking for Free Tier]
    FreeCheck -->|Yes| PendingApp[Status: Pending Approval from Organizer]
    
    %% Instant Match (Locco Token)
    TypeCheck -->|Instant / Locco-Match| TokenCheck{Wallet Balance >= 1?}
    TokenCheck -->|No| TokenReject[Reject: Insufficient Locco Tokens]
    TokenCheck -->|Yes| DeductToken[Deduct 1 Token from Wallet]
    DeductToken --> AutoApprove

    %% Organizer Removal Logic
    Cancel[Organizer tries to remove Participant] --> TimeCheck{Hours until event < 3?}
    TimeCheck -->|Yes| BlockCancel[Reject: Cannot remove < 3h before start]
    TimeCheck -->|No| ConfCancel[Participant Removed]
```

### B. RunBuddy Matching Logic (`runBuddyStore`)

Definisce la logica dello swipe e dello sblocco chat basata sulla dimensione del gruppo bersaglio.

```mermaid
flowchart TD
    StartSwipe[User views RunBuddy Profile] --> SwipeAction{Swipe Left or Right?}
    
    SwipeAction -->|Left (Skip)| NextProfile[Show Next Candidate]
    
    SwipeAction -->|Right (Like)| LimitCheck{Has Swipe Quota? \n Max 5/day Free}
    LimitCheck -->|No| BlockSwipe[Prompt Premium Upgrade]
    LimitCheck -->|Yes| DeductSwipe[Register Swipe]

    DeductSwipe --> MatchAlgo{Algorithm Match? \n e.g., 50% chance mock}
    MatchAlgo -->|No Match| NextProfile
    MatchAlgo -->|Match| AddMatch[Add to Temp Matches Pool]
    
    AddMatch --> GroupCheck{Temp Matches >= \n targetGroupSize? \n Default: 3}
    GroupCheck -->|No| AwaitGroup[Wait for more matches]
    GroupCheck -->|Yes| UnlockChat[Unlock Group Chat for all matched buddies]
```

### C. Check-In & Game Economy

Gestisce cosa succede quando si completa e si valuta un'attività.

```mermaid
flowchart LR
    StartEvent[Event Starts] --> DoCheckIn[User does Check-In \n via GPS or Organizer]
    DoCheckIn --> EventEnd[Event Finished]
    EventEnd --> TriggerReview[Review Flow Triggered]
    TriggerReview --> RateUsers[Rate Punctuality, Level, Friendliness]
    RateUsers --> EconomyReward[Reward Systems: \n Reliability Score Updated \n Wallet Tokens Added]
```

---

## 3. Deep Dive: RunBuddy & Chat UI Flow

Questo diagramma esplode in dettaglio il flusso dell'utente (UI Flow) partendo dal discovery tramite Swipe fino all'organizzazione della corsa in chat, coprendo `runbuddy.tsx`, `messages.tsx` e `chat/[id].tsx`.

```mermaid
flowchart TD
    %% RunBuddy Swipe Interface
    RB_Start[Tab: RunBuddy] --> RB_Feed[Load Profiles & Ads Feed]
    RB_Feed --> RB_Action{User Action}
    
    RB_Action -->|Swipe Left| RB_Next[Next Profile]
    
    RB_Action -->|Swipe Right| RB_CheckLimits{Check Limits/Auth}
    RB_CheckLimits -->|Limit Reached| RB_ModalPremium[Show Premium Upgrade Modal]
    RB_CheckLimits -->|Allowed| RB_SwipeApi[Handle Swipe & Check Match]

    RB_SwipeApi -->|No Immediate Match| RB_Toast[Send Request \n Silent]
    RB_SwipeApi -->|Match!| RB_ShoeAnim[Flying Shoe Animation]
    
    RB_ShoeAnim --> RB_MatchModal[Show 'Loc-me!' Modal]
    RB_MatchModal -->|Start Chat| Nav_Messages[Navigate to Messages Tab]
    RB_MatchModal -->|Keep Searching| RB_Next

    %% Messages Hub
    Nav_Messages --> Msg_Hub[Tab: Messages - Activity Hub]
    
    Msg_Hub --> Msg_Upcoming[View Upcoming Events]
    Msg_Upcoming --> Nav_RunDetail[Navigate to Run Detail]
    
    Msg_Hub --> Msg_Pending[View Pending Match Requests]
    Msg_Pending -->|Quick Accept| Msg_ActionAccept[Update Status -> Active]
    Msg_Pending -->|Quick Decline| Msg_ActionDecline[Archive Chat]
    Msg_Pending -->|Tap Item| Nav_ChatPending[Navigate to Chat \n State: Pending]

    Msg_Hub --> Msg_Active[View Active Chats]
    Msg_Active -->|Tap Item| Nav_ChatActive[Navigate to Chat \n State: Active]

    %% Chat Interface
    Nav_ChatPending --> Chat_Profile[Show Profile Overview \n with Accept/Decline]
    Chat_Profile -->|Accept| Chat_ActiveState[Switch to Active Chat View]
    Chat_Profile -->|Decline| Chat_Exit[Archive & Go Back]

    Nav_ChatActive --> Chat_ActiveState
    
    Chat_ActiveState --> Chat_TextInput[Typing standard message]
    Chat_ActiveState --> Chat_Propose[Tap Proposal Button '+']
    
    Chat_Propose --> Prop_Modal[Show Proposal Modal \n Time & Location]
    Prop_Modal --> Prop_Send[Send Proposal]
    Prop_Send --> Chat_InlineCard[Render Proposal Card in Chat]
    
    %% Locco Place Integration
    Prop_Send -.-> Fetch_Locco[API: Get Locco Place Suggestion]
    Fetch_Locco --> Locco_PlaceCard[Render Suggested Partner/Place Card]
```
