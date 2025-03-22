# Dating App Design - Hinge-like App

## 1. Overview

This is a system design for a dating app similar to **Hinge**. The app focuses on personalized user engagement with profile creation, media uploads, swiping, matching, and messaging. The system is designed to be scalable, available, and consistent, ensuring a smooth and fast experience for users worldwide.

---

## 2. Functional Requirements

### 2.1 User Profile Management
- Users can create and update profiles with basic information (name, age, interests, etc.).
- Users can upload photos and videos (with a 5GB maximum size for videos).
- Users can update preferences for age range, interests, etc.

### 2.2 News Feed
- Personalized recommendations for other users based on location, preferences, and past interactions.
- Users can like or comment on profiles, influencing future recommendations.

### 2.3 Swiping
- Users swipe right for interest and left for no interest.
- If both users swipe right, a match occurs.

### 2.4 Messaging
- Only matched users can send messages to each other.
- Messages are sent in real-time using WebSocket for instant communication.

### 2.5 Unmatch/Block
- Users can unmatch or block someone, which removes them from the match list and prevents further interaction.

---

## 3. Non-Functional Requirements

### 3.1 Scalability
- The system must support a large number of users, potentially scaling to a global user base.

### 3.2 Availability
- Ensure high availability of the news feed and messaging features with minimal downtime.

### 3.3 Consistency
- The match-making process should be consistent, ensuring reliable match notifications and conversations.

### 3.4 Performance
- The app should provide quick access to user profiles and media, ensuring a smooth user experience under high traffic.

---

## 4. System Architecture

### 4.1 High-Level Architecture (HLD)

#### 4.1.1 Frontend (Mobile/Web App)
- The client app communicates with the backend API to access user profiles, swipes, and matches.

#### 4.1.2 Load Balancer
- Distributes incoming traffic across multiple web servers to handle high loads.

#### 4.1.3 Web Servers
- Hosts the backend logic and processes requests for profile management, swiping, and messaging.

#### 4.1.4 Cache (Redis)
- Caches frequently accessed data to reduce database load and improve performance.

#### 4.1.5 Databases
- **NoSQL Database (e.g., Cassandra/DynamoDB):** Stores user profiles, swipes, matches, and interactions.
- **SQL Database:** Used for transactional data (e.g., block/unmatch information).
- **Search Engine (e.g., Elasticsearch):** Powers fast profile lookups and search queries.

#### 4.1.6 Media Storage
- **Amazon S3** or **Google Cloud Storage** for storing user photos and videos.
- **Content Delivery Network (CDN):** Delivers media content globally with low latency.

#### 4.1.7 Real-Time Messaging
- **WebSocket** for real-time bidirectional communication between matched users.
- **Message Queue (Kafka/RabbitMQ):** Ensures reliable message delivery.

#### 4.1.8 Recommendation Engine
- A machine learning-powered engine that suggests profiles based on user interactions, preferences, and location.

#### 4.1.9 User Analytics
- Collects data on user interactions to improve the recommendation engine and user experience.

---

## 5. Detailed Component Design

### 5.1 User Profiles

- **Profile Data:**
  - Contains personal information, photos, videos, preferences, and matches.
  
- **Database Schema:**
  - **User**: (ID, name, age, location, preferences)
  - **Media**: (ID, type, URL, userID)
  - **Match**: (user1ID, user2ID, match_created_at)

### 5.2 Swiping and Matching

- **Swipe Data:**
  - Records user swipes (right/left) on other users.
  - When both users swipe right, a match is created.

- **Match Database Schema:**
  - **Match**: (user1ID, user2ID, match_created_at)
  - **Swipe**: (userID, targetUserID, swipe_direction, timestamp)

### 5.3 Messaging

- **Real-Time Messaging:**
  - WebSocket connections allow matched users to communicate in real-time.
  
- **Message Database Schema:**
  - **Message**: (senderID, receiverID, content, timestamp)

### 5.4 Media Uploads

- **Media Storage:**
  - Users can upload photos and videos (max 5GB for videos).
  - Stored in **cloud storage** (e.g., Amazon S3, Google Cloud Storage).

- **Media Database Schema:**
  - **Media**: (ID, userID, type, URL, size)

---

## 6. Back of the Envelope Estimations

### 6.1 Estimated User Base and Traffic

- **Assumptions:**
  - Estimated number of daily active users (DAU): 50 million.
  - Total monthly active users (MAU): 200 million.
  - Average number of swipes per user per day: 100 swipes.
  - Average number of messages per match per day: 20 messages.

### 6.2 Data Size Estimates

#### 6.2.1 User Profile Data

- **Data per user:**
  - Basic profile data (e.g., name, age, preferences): 1 KB.
  - Photos/videos (assuming 5 photos per user, 1MB each): 5MB per user.
  - Total profile data for 200 million users:
    - Profile data: 200 million users * 1 KB = 200 MB
    - Photos/Videos: 200 million users * 5MB = 1,000 TB

#### 6.2.2 Swipes

- **Swipe Data per user:**
  - Swipe data per day: 100 swipes.
  - Data per swipe: 100 bytes (userID, targetUserID, timestamp).
  - Daily swipe data for 50 million DAU:
    - Daily swipe data: 50 million users * 100 swipes = 5 billion swipes/day
    - Data per day: 5 billion swipes * 100 bytes = 500 GB/day
  - Monthly swipe data: 500 GB/day * 30 days = 15 TB/month

#### 6.2.3 Messages

- **Message Data per match:**
  - Average message size: 200 bytes.
  - Average number of messages per match: 20 messages.
  - Data per match: 20 messages * 200 bytes = 4,000 bytes (4 KB) per match.
  - Total data for 50 million DAU:
    - Total matches per day: 50 million DAU * 0.5 match per user = 25 million matches/day.
    - Daily message data: 25 million matches * 4 KB = 100 GB/day.
  - Monthly message data: 100 GB/day * 30 days = 3 TB/month.

### 6.3 Throughput Estimations

#### 6.3.1 Swiping Throughput

- **Swipes per second:**
  - With 50 million DAU and 100 swipes per user per day:
  - Swipes per day: 50 million * 100 = 5 billion swipes/day.
  - Swipes per second: 5 billion swipes/day / (24 * 60 * 60) = ~57,870 swipes/sec.

#### 6.3.2 Messaging Throughput

- **Messages per second:**
  - With 25 million matches/day and 20 messages per match:
  - Messages per day: 25 million * 20 = 500 million messages/day.
  - Messages per second: 500 million messages/day / (24 * 60 * 60) = ~5,786 messages/sec.

---

## 7. API Endpoints and WebSocket Routes

### 7.1 API Endpoints

- **POST /users**  
  Register a new user.

- **POST /auth/login**  
  Login a user.

- **GET /users/{userId}/profile**  
  Retrieve the profile of a specific user.

- **PUT /users/{userId}/profile**  
  Update the profile of a specific user.

- **GET /users/{userId}/recommendations**  
  Retrieve profile recommendations for a specific user.

- **POST /swipes**  
  Swipe on a profile (right/left).

- **GET /matches/{userId}**  
  Retrieve the matches for a specific user.

- **POST /messages**  
  Send a message to a match.

- **GET /messages/{userId}**  
  Retrieve all messages for a user with a specific match.

### 7.2 WebSocket Routes

- **/ws/connection**  
  Establish a WebSocket connection for real-time communication.

- **/ws/messages/{userId}**  
  Send/receive messages to/from a match via WebSocket.

- **/ws/notifications/{userId}**  
  Real-time notifications for new matches or messages.

---

