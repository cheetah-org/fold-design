graph TD
    Woman((Woman))
    Man((Man))
    Admin((Admin))
    
    Woman -.-> |Registers, Browses, Likes, Chats| DatingApp[Dating App]
    Man -.-> |Registers, Receives Likes, Chats| DatingApp[Dating App]
    Admin -.-> |Manages, Monitors| DatingApp[Dating App]
    
    DatingApp -.-> |Sends OTP| SMS[SMS Gateway] 
    DatingApp -.-> |Sends Push| Push[Push Notification Service]
    DatingApp -.-> |Uploads| AppStore[App Stores]
    DatingApp -.-> |Reports| Analytics[Analytics Platform]
    
    DatingApp -.-> |Verifies Gov ID| IDVerification[ID Verification Service]
