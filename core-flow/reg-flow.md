sequenceDiagram
    participant User as User
    participant Client as Mobile Client
    participant Auth as Auth Service
    participant Ratio as Ratio Service
    participant User as User Service
    participant DB as Database

    User ->> Client: Enter DOB, phone number
    Client ->> Auth: Send OTP request
    Auth ->> DB: Check if user exists
    Auth ->> SMS: Send OTP
    User ->> Client: Enter OTP
    Client ->> Auth: Verify OTP
    Auth ->> Ratio: Check gender ratio
    alt If woman
        Ratio ->> Auth: Ratio OK, proceed
        Auth ->> DB: Create user
        Auth ->> Client: Registration successful
    else If man and ratio OK
        Ratio ->> Auth: Ratio OK, proceed 
        Auth ->> DB: Create user
        Auth ->> Client: Registration successful
    else If man and ratio not OK
        Ratio ->> Auth: Ratio not OK, queue
        Auth ->> DB: Create queued user
        Auth ->> Client: Added to queue
        loop 
            Ratio ->> Ratio: Check ratio
            Ratio ->> DB: Update queue
            Ratio ->> Client: Queue update notification
        end
        Ratio ->> Auth: Ratio OK, admit from queue
        Auth ->> DB: Update user status
        Auth ->> Client: Registration successful
    end