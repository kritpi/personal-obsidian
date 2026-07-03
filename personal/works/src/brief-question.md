### Agenda
- Can this person solve engineering problems?
- Can this person explain systems they built?
- Can this person work as a team?
- Does this person have a growth mindset?
- Can this person survive?

### Introduction (STAR + Soft Skills)
- Hi, I'm Kritpiruch (Amp)
- recent grad CS KU
- (**Situation**) Software Engineer Internship experience
	- Central Food Retail - Go Wholesale
	- Working with Golang, Postgres, Microservices, GCP (optional)
- (**Task**) Responsible for
	- Design & Build backend services
		- Channel API 
			- Connect between app and core services(order, payment)
		- CMS API
			- API for CMS portal used for updating App widgets, content
		- App features
	- Collaborating with business, frontend, others teams
		- BA, PO, UX/UI, QA
- (**Action**) Notification scheduling platform, others features
	- Coordinate with business team
		- Sprint grooming
		- Understanding features and providing feedback
	- Planning features, testing, production support
- (**Result**) Features shipped to production 
	- Deployment (env DEV, SIT, UAT, PROD)
	- L2 support after deployment (find root cause, fix)
#### Strength
- Strategic & Structured Thinking - มองภาพรวม process, วางแผนเป็นขั้นตอน
- Systematic, but creativity - ชอบงานที่ core structure ชัดเจน แต่ใช้ creativity adapt ได้มีประสิทธิภาพ
- Empathetic, Team Player - ใจเย็น รับฟังมุมมองคนในทีม
#### Weakness
- Lack in details execution - หลุดโฟกัสกับรายละเอียดเล็กๆ (ทำ checklist ก่อนที่จะเริ่ม implement)
- Self-Promotion - public speaking ไม่ค่อยเก่ง, energy management ทำได้ไม่ดี อาจจะมีช่วงที่ work ได้เต็มที่ แต่ก็จะมีบางช่วงที่รู้สึกว่าต้องพักบ้าง
- People pleaser - ขอความช่วยเหลือไม่เก่ง บางทีถืองานที่ยาก หรือว่าเยอะเกินตัว แล้ว handle ไม่ได้ (ประเมินตัวเองก่อนที่จะขอความช่วยเหลือช้าเกินไป)

### Skills (Deep Dive)
- Backend Microservices Architecture
- Cloud Platform (GCP)
	- Cloud Run (Serverless, request driven)
	- Cloud Run Jobs (Task driven, batch processing)
	- Pub/Sub (many-to-many streaming, topic, background process, fan-out architecture)
	- Cloud Tasks (one-to-one, create task, specific endpoint, scheduling)
		- can be scheduled 30 days in advance
		- 24 hours deduplication + 24 hours timeout (cloud task wait for response after execution)
	- Eventarc (event routing)
		- event driven (listen for changes eg. listen to file upload)

### Work Experience Deep Dive (Go Wholesale)
#### Notification Scheduler
- Setup scheduler from **CMS portal**
	- Name, description, start date, end date, sent time, recurring, selected date, audiences
	- eg. `every friday 12:30 from 2025-07-11 to 2026-07-11`
%% - Request go to **BFF** service
	- Route request from CMS through each service + JWT extraction %%
- **Notification Set-up service**
	- Validate request 
	- Write to db (postgres)
- **Cloud Run Jobs** 
	- Every midnight 
	- Reads which notifications to sent today
	- Schedule each notification to **Cloud Task**
- **Cloud Task**
	- Scheduled with sent time
	- `notification_id` as json payload
	- Ensure idempotency, cloud task cannot schedule deduplicate key in case of cloud run job retry
	- Call API to sent notification
		- Query notification data
		- Publish to Pub/Sub
- **Notification Process Service**
	- Received message from Pub/Sub
	- Find audience segment (member_id : FCM token)
	- Sent notification using FCM token
**TL;DR**
- API for notification configuration
- Cron job(daily) query notifications to be sent from db
- Schedule each notification to cloud task
- Notification **publish** to Pub/Sub
- Sending service **received** message, and sent notification (FCM token)
#### Notification Go Bonus Targeted
- Sent notification to customer whose joined the campaign, with x thb remaining purchase amount before reaching next tier
- **Extended** from notification scheduler feature
- Select Go Bonus campaign to target
- Choose remaining purchase amount
- Query heavy
#### Go Bonus (Accumulate Campaign)
- Tier-based rewards campaign where customers accumulate purchase amounts to unlock coupon rewards.
- CMS setup campaign
	- Campaign data (name, start, end date, etc.)
	- Setup tiers and coupons (50 tiers, 3 coupon/tier)
- Display as a page + widget on Go App
- Display promoted products
- Accumulate purchase amount after order created
#### Order History
- Provide order history functionality for future Reorder support
	- Group purchased product
		- Group by order
		- Group by product category
- Multiple service/API data retrieval
	- Order history (SKUs, amount, total price)
	- Product Category
	- Product Details (api support multiple products details retrieval)
		- Use Goroutine for a batch retrieval concurrently
		- Each order could contain a lot of products
	- Get products in customer cart (firestore)
- Aggregation
	- Sorting order (latest to oldest)
	- Sorting category
	- Create a unique SKUs list before getting product details
	- Group by order/category
**Challenging:** 
- optimization to avoid nested loop
- sync product details received from Goroutine
#### Go Mail
- Personalized, customer segment based promotion features for displaying promotion products
- Aggregate data from 2 sources
	- products setup on CMS
	- personalized products from Data team (BigQuery)
- IE, IA, IR (Item Engage, Item Active, Item Repurchase)
	- IA - Just purchase
	- IE - Have purchased (~30 days)
	- IR - Not repurchase
- Personalized data updated everyday
	- Cache product SKUs on first time user visit promotion page
	- 1 day TTL
#### Product Badge
- Feature for displaying product badges
- Integrated with upstream services
- Enhancing existing APIs to support changes
	- Upstream services integration
	- Badge display on app
		- Search page
		- Product listing
- Filter some products category out (Alcohol)
### Technical Question
#### Authentication + Authorization
- Authentication - Who are you
- Authorization - What are you allowed to do
- JWT (json web token)
	- header (alg, typ)
	- payload (eg. name, exp, sub, iat)
	- signature
- Access token
	- short lived
	- identity, permission
	- sent with API request
- Refresh token
	- long lived
	- generate new access token
- Storing token
	- Access token - in client memory (eg. useState), but not persistance (or stored in cookie)
	- Refresh token - Cookie(implemented with in-memory access token)
#### OAuth2.0
- Allow using 3rd party applications to use as authentication instead of using username, password
- Google Login flow
	- Client redirect to authorization page (client_id, callback url)
	- Google redirect back with `authorization_code`
	- Client sends `authorization_code` to server
	- Server exchange code for token 
		- calls Google token exchange endpoint
		- `client_id, client_secret, authorization_code`
	- Server receive `access_token`, `id_token(JWT)`, `refresh_token`
#### Git
- Merge and Rebase
	- Merge - merge feature into master branch (1 commit)
	- Rebase - relocate whole feature's commits into head of master branch

**Agile**
- 2 weeks/sprint
- Planning & Estimation
	- Planning poker
- Development Process
	- Dev, test
- Sprint Grooming (Backlog refinement)
	- Planning for next sprint with Business
	- System designing, API specs
- Retrospective
	- What went well
	- Learn what
	- What to be improve
	- Appreciation
- Deployment timeline