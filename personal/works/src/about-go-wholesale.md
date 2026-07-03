**Overview**
- channal api for an aggregate data from another services (product, payment, orders, etc.)
- work with frontend application teams + CMS portal
- use golang with google cloud, posgres and firestore

**Notification** 
- CMS setup -> BFF -> Notification service
- notification data in postgres (day of week, start, end, sent time, month, day, interval, is-sent)
- job every midnight schedule notification (which to be sent in that day)
- job also schedule cloud task (each notification)
- cloud task call send notification api (toggle is-sent flag, publish message to pub/sub)
- another notification service subscribe to pub/sub topic -> get fcm token -> sent notification
---
notification with accumulate campaign
- targeted campaign with price target
- query customer in campaign that have total purchase value near the target price
- sent notification

**Order History**
- get user's latest order history (outbound service)
- extract product SKUs from order history (unique)
- use Goroutine to get product details from the SKUs (outbound service)
- get product category (outbound service)
- get product in cart
- map products details back to each order (with amount of product in cart)
- map products details to product categories

**Go Mail**
- personalized(customer segment) product catalog features
- promotioned SKUs uploaded from CMS (product IA, IE, IR)
- uploaded product merged with product reccommendation (from data team)
- use Redis to cache a data from preload api (first api called to paged of the day after data team reset the reccommended product)
- preload api return metadata of SKUs in each product segments
- get product details to show getting SKUs from Redis

**Go Bonus**
- purchase based reward campaign system
- set up using CMS by creating campaign with tiers
- 1 campaign have 50 tiers
- 1 tier have coupon rewards

**Product Badge**
- enhance api to support product badge (outbound service)
- enhance api in pages to show badge (search, product card, etc.)

## Related
- [[interview-prep|Interview Prep]]
