# Shopify Interview & Implementation Questions

---

## General Shopify API Questions

1. What are the main differences between the **REST Admin API** and **GraphQL Admin API**?
2. Which API would you choose for building a dashboard and why?
3. What types of Shopify APIs are available and when would you use each (Admin API, Storefront API, etc.)?
4. What is the purpose of **API scopes**, and how do you configure them?
5. How do you handle **API versioning** in Shopify?

---

## Authentication & Authorization

6. How does Shopify's **OAuth 2.0 flow** work?
7. How do you securely store **access tokens** after installation?
8. What are **offline tokens** vs **online tokens** in Shopify?
9. How do you ensure your app doesn't lose access when a token expires?
10. How would you implement authentication for multiple stores in a single app?
11. How did you securely authenticate with Shopify's API?
12. How do you get HMAC key in Shopify?
13. Where do you store access tokens? Did you secure the webhook endpoints?

---

## Shopify Admin API (REST & GraphQL)

14. How do you retrieve orders using the Admin API?
15. How would you query only fulfilled/unfulfilled orders?
16. How do you retrieve paginated data (e.g., 10,000+ orders)?
17. What are the best practices for minimizing API calls?
18. How do you calculate metrics like **average order value (AOV)** or **customer lifetime value (CLTV)** using Shopify data?
19. What GraphQL queries have you worked on?

---

## Webhooks & Real-Time Updates

20. What is a webhook in Shopify and how do you register one?
21. How would you verify that a webhook request came from Shopify?
22. What types of events can you subscribe to?
23. What do you do if your webhook delivery fails?
24. How do you handle webhook retries?
25. How will you know if the order is fulfilled? Using Webhooks
26. When order is created and you create a webhook, and when it is triggered, how do you update the order status?
27. How do you ensure your dashboard stays up to date?

---

## Rate Limiting & Error Handling

28. How does Shopify handle rate limiting in the REST and GraphQL APIs?
29. What headers should you check to monitor rate limits?
30. How do you throttle requests in your code to avoid being rate limited?
31. How do you handle 429 (Too Many Requests) responses?
32. What would you do if Shopify's API is temporarily down?
33. How did you handle Shopify's API rate limits?
34. What happens if the API request fails or the webhook fails to deliver?

---

## App Development

35. How do you structure a Shopify app to support multiple stores?
36. What is the Shopify App Bridge, and when would you use it?
37. How do you embed your app inside the Shopify Admin?
38. How do you handle app uninstallation cleanup?
39. How do you update your app when Shopify releases a new API version?
40. Describe your experience with Private/Public apps vs Custom apps
41. Share your experience with Node.js for writing APIs

---

## Checkout & Order Flow

42. Create a flow of order creation in Shopify
43. What are Checkout UI Extensions?
44. In checkout extension, how can you get input of cart items?
45. What are checkout blocks?
46. What are customer events?
47. What are Pixels in Shopify and how are they used?

---

## Product & Storefront Customization

48. How do you connect a video file with a product?
49. How can you show products with multiple colors as individual products in search result in PLP?
50. What is theme customization? What options do you have if no extension is available?
51. If you want to hide all products which are out of stock, how would you do it?
52. If you need to create different PDP layouts for different markets, how would you implement this in a multi-market landscape?
53. How would you fetch product data for a headless storefront?
54. What is the **Storefront API**, and how does it differ from Admin API?
55. How do you create custom checkout flows with Shopify?
56. What are Shopify Functions, and when would you use them?
57. Can you use Shopify APIs to apply discounts or modify carts?

---

## Security & Best Practices

58. How do you protect your app from **CSRF** and **XSS**?
59. How do you protect webhook endpoints from tampering?
60. How do you store customer or order data securely?
61. What are some Shopify API usage anti-patterns?
62. How do you implement retry logic for failed API requests?

---

## Testing & Deployment

63. How do you test your Shopify app without affecting a live store?
64. How do you test webhook behavior during development?
65. What Shopify CLI tools have you used for development?
66. How do you deploy updates to your Shopify app with minimal downtime?
67. How would you debug API errors in a production app?

---

## Real-World Application Logic

68. How would you build a revenue dashboard using Shopify data?
69. How can you track best-selling products over time?
70. How would you build a system to detect unusual order volume spikes?
71. What are ways to segment customers (e.g., by purchase behavior)?
72. How would you automatically tag high-value customers?

---

## Advanced Shopify Implementations

73. Details on past projects you've worked on?
74. B2B Implementation on Shopify
75. Market setup
76. Advanced shipping setup
77. Multi-currency, language switcher implementation
78. Upsell implementation
79. Cross-sell implementation
80. Advantage of Shopify Plus
81. Multi-location setup
