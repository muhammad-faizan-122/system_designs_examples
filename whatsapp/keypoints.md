The system design for a chat application like WhatsApp, as explained by Gaurav Sen, involves a complex but highly scalable architecture. Here's a breakdown of the core components and the flow of messages through the system. 

 ### **Core Components of the System** 

 **1. Gateways:** 
 *   **Function:** Gateways are the entry point for all client connections to the WhatsApp backend. When a user opens the app, their device establishes a persistent connection, ideally using WebSockets, with a specific gateway server. 
 *   **Details:** These servers are designed to be "dumb," meaning they don't hold much business logic. Their primary role is to maintain the connection with the user's device and pass messages between the client and the backend services. This design helps to minimize the memory footprint on the gateway servers, which is crucial for handling millions of concurrent connections. 

 **2. Load Balancer:** 
 *   **Function:** The load balancer distributes incoming connection requests from user devices across the available gateway servers. 
 *   **Details:** This ensures that no single gateway server becomes overwhelmed with traffic, which helps in maintaining high availability and reliability. The load balancer can use various algorithms, such as round-robin or least connections, to decide which gateway should handle a new connection. 

 **3. Sessions Service:** 
 *   **Function:** This is a crucial microservice that keeps track of which user is connected to which gateway server. It essentially maintains a mapping of `UserID` to `GatewayID`. 
 *   **Details:** By decoupling this "session" information from the gateways, the system becomes more scalable and resilient. The gateways themselves don't need to know where other users are connected; they can simply query the Sessions Service to route a message. 

 **4. Group Service:** 
 *   **Function:** This service manages all the information related to groups, such as group members and group metadata. 
 *   **Details:** When a message is sent to a group, the Sessions Service contacts the Group Service to get the list of all members in that group. This separation of concerns keeps the Sessions Service focused on individual user connections while the Group Service handles the complexities of group management. 

 **5. Auth Service:** 
 *   **Function:** The Authentication Service is responsible for verifying the identity of users when they connect to the system. 
 *   **Details:** While not deeply detailed in the provided explanation, a typical auth service would handle user login, token generation, and validation to ensure that communication is secure and only authorized users can access the system. 

 **6. Message Queues:** 
 *   **Function:** Message queues are used to handle the asynchronous processing of messages and to ensure message delivery even if a service is temporarily unavailable. 
 *   **Details:** When a message is sent, it can be placed in a queue. A consumer service then picks up the message and processes it. This is particularly useful for features that can be deprioritized during peak loads, like updating the "last seen" status. 

 ### **Message Flow Explained** 

 #### **One-to-One Chat** 

 1.  **Connection:** User A opens WhatsApp and their device connects to a Gateway (let's say Gateway 1) via a WebSocket connection, a process managed by the Load Balancer. The Sessions Service records that User A is connected to Gateway 1. Similarly, User B is connected to Gateway 2, and this is also recorded. 

 2.  **Sending a Message:** User A sends a message to User B. The message travels through the established WebSocket to Gateway 1. 

 3.  **Routing:** Gateway 1, being a "dumb" terminal, forwards the message to the Sessions Service. 

 4.  **Recipient Location:** The Sessions Service looks up User B's session information and finds that they are connected to Gateway 2. 

 5.  **Message Delivery:** The Sessions Service then routes the message to Gateway 2. 

 6.  **Receiving the Message:** Gateway 2 sends the message down the WebSocket connection to User B's device. 

 #### **Sent, Delivered, and Read Receipts** 

 *   **Sent (Single Tick):** When User A's message reaches the Sessions Service and is persisted (e.g., stored in a database to ensure it won't be lost), the Sessions Service sends an acknowledgment back to Gateway 1, which then informs User A's device that the message has been sent. 

 *   **Delivered (Double Tick):** After User B's device receives the message from Gateway 2, it sends an acknowledgment back up to Gateway 2. This acknowledgment travels back to the Sessions Service. The Sessions Service then looks up User A's location (Gateway 1) and sends a "delivered" notification. 

 *   **Read (Blue Ticks):** When User B opens the chat and sees the message, their device sends a "read" event. This event follows the same path as the "delivered" receipt, ultimately resulting in User A's app displaying the blue ticks. 

 #### **Group Chat** 

 1.  **Sending a Group Message:** A user (from the "red" group) connected to Gateway 1 sends a message to the group. 

 2.  **Initial Routing:** The message goes from Gateway 1 to the Sessions Service. 

 3.  **Fetching Group Members:** The Sessions Service, realizing it's a group message, queries the Group Service with the `GroupID`. 

 4.  **Member List:** The Group Service returns a list of all `UserID`s who are members of that group. 

 5.  **Fanning Out the Message:** The Sessions Service then looks up the gateway connection for each member in the list. It finds that other members of the "red" group are connected to Gateway 2. 

 6.  **Delivering to All Members:** The Sessions Service sends the message to all the respective gateways, which then deliver it to the individual group members' devices. To handle a large number of members in a group, WhatsApp limits the group size to prevent the system from being overwhelmed by this "fan-out" process.