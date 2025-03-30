# Handling Peak Load on Goal.com — Architectural Ideas and Optimizations

> 💬 I did a quick research on [goal.com](https://www.goal.com/en) and found several areas that could potentially be improved.  
> I tried to identify which pages might experience peak loads and what architecture would handle them effectively.

---

## 1. Real-time Updates for a Large Number of Users

### 🧑‍💻 User story:
A user visits the [live matches page](https://www.goal.com/en/live-scores) and expects the score to update automatically, without refreshing the page or clicking anything.

### 🔹 Problem:
- If each user polls the API every second or every 5 seconds, the server receives thousands of simultaneous requests.
- This creates load on the API, the database, and the network.

### ✅ Solution:
- Implement **Server-Sent Events (SSE)** or **WebSocket** so that the server sends updates only when they actually occur.
- Use an event-driven architecture, where the business logic service sends messages to a queue (e.g., RabbitMQ), and the WebSocket/SSE servers broadcast them to the frontend.
- This way, we:
  - reduce API requests;
  - ensure scalability (WS servers can be scaled horizontally);
  - avoid overloading the main API.

### 💡 Potential Improvements:
1. **Auto-scaling WebSocket/SSE servers** before major matches (based on schedule or traffic forecasts) to support a large number of connections.
2. **Batching updates** for pages listing all matches of the day — collect updates over the last 10–15 seconds and send them in one batch.

---

## 2. Loading Large Amounts of Static Content

### 🧑‍💻 User story:
A user opens the [home page](https://www.goal.com/en/player/cristiano-ronaldo/h17s3qts1dz1zqjw19jazzkl) or a [news article page](https://www.goal.com/en/news) and waits for banners, player images, avatars, etc., to load.

### 🔹 Problem:
- A large number of images.
- If all these files are served from the main server — slow loading, low performance, and unnecessary backend load.

### ✅ Solution:
- Use a **CDN** located geographically closer to the user (e.g., Cloudflare, Akamai).
- All images should be cached in the CDN.
- Add long-term caching policy (e.g., `Cache-Control: max-age=...`).

---

## 3. Text Content, Articles (News, Analysis)

### 🧑‍💻 User story:
A user reads news, browses through categories (e.g., [Transfer News](https://www.goal.com/en/transfer-news)). If subscribed — they expect notifications about new articles.

### 🔹 Problem:
- Every new article updates category lists and triggers notifications for many users.

### ✅ Solution:
- Organize reading lists via cache (e.g., Redis) by categories (latest 20 publications).
- When adding a new article — **append it to the cached list**, without resetting the entire cache.
  - If an article is deleted or updated — logic should sync changes with the cache.
- Send notifications not immediately to all:
  - launch a **task queue** that gradually sends messages (e.g., 1,000 users at a time).

---

## 4. Statistics Pages (Players, Teams, Seasons)

### 🧑‍💻 User story:
A user opens a [player page](https://www.goal.com/en/player/erling-haaland/) to view their season stats, or a team stats page.

### 🔹 Problem:
- Stats are often generated using complex database queries.
- Data from past seasons doesn’t change but is recalculated every time.

### ✅ Solution:
- Store past season data in **denormalized form** to avoid recalculating each time.
- Update current season data incrementally (only what has changed).
