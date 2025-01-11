# E-commerce App (like Amazon)

*scale: 1M daily active users(browsing and buying)*


- [E-commerce App (like Amazon)](#e-commerce-app-like-amazon)
  - [Functional Requirements](#functional-requirements)
    - [Out of scope](#out-of-scope)
  - [Non Functional Requirements](#non-functional-requirements)
  - [Entities](#entities)
  - [Api or Interfaces](#api-or-interfaces)
  - [High Level Design](#high-level-design)
  - [Deep Dives](#deep-dives)
    - [One User One Product Consistency](#one-user-one-product-consistency)
    - [Reduce Search Latency](#reduce-search-latency)
    - [High Availability by Scaling](#high-availability-by-scaling)
    - [Handle Purchase Surges](#handle-purchase-surges)


## Functional Requirements
- Searching for product 
- Viewing details of product
- Purchasing the product
  - Making the payment ( online payment is the only option)
  - Order placed(for successful payment)
- User can track the order

### Out of scope

- Add product to cart 
- Customer support


## Non Functional Requirements

- Consistency in terms of purchasing product i.e one user one product Id
- Availability in terms of searching of items, viewing details adding to cart(out of scope)
- Low search latency > 1s
- Handle huge purchase surges like new phone dropped


## Entities

- Product
- Seller
- Order
- PaymentReceipt
- User

## Api or Interfaces

***Header: JWT | session token for authentication***

**Search for product**

```
GET /search/?term={term} ----> PartList<Product>
```
**Product Details**
```
GET /product/{productId}/?pagination={pages}?size={size}
```

**Reserve the product for purchase**
```
POST /product/purchase---> redirected to payment page
body: {
    productId
}
```
**Make payment**

```
POST /product/payment ----> Order and Receipt
body:{
    productId
    paymentDetails
}
```
**Track order**

```
GET /track/{orderId}----> Order detail
```

## High Level Design

![high-level design](image.png)

## Deep Dives

### One User One Product Consistency
### Reduce Search Latency
### High Availability by Scaling
### Handle Purchase Surges

![deep dives](image-1.png)

