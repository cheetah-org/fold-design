sequenceDiagram
    participant Woman as Woman
    participant Client as Mobile Client
    participant Matching as Matching Service
    participant User as User Service
    participant Notif as Notification Service
    participant Man as Man
    participant DB as Database

    Woman ->> Client: Browse feed
    Client ->> Matching: Get profiles
    Matching ->> DB: Get eligible men profiles
    DB -->> Matching: Profiles
    Matching -->> Client: Feed profiles
    Client -->> Woman: Display feed
    Woman ->> Client: Like profile
    Client ->> Matching: Create like
    Matching ->> DB: Save like
    Matching ->> Notif: Send push notification
    Notif ->> Man: New like notification
    Man ->> Client: View like
    Client ->> Matching: Get like details
    Matching ->> DB: Get like and woman's profile
    DB -->> Matching: Like and profile
    Matching -->> Client: Like details
    Client -->> Man: Display woman's profile
    Man ->> Client: Accept like
    Client ->> Matching: Accept like
    Matching ->> DB: Create match, update like status
    Matching ->> Notif: Send match notifications
    Notif ->> Woman: Match notification
    Notif ->> Man: Match notification