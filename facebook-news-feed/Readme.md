# FaceBook News Feed

[Requirements](#requirements)

[Design constraint](#design-constraint)

[Core entities](#core-entities)

[Api or interfaces](#api-or-interface)

[High level design](#high-level-design)




## Requirements

**Functional**:

- User should be able to post (new post)
- User should be able to follow (follow)
- User should be able to see feed of posts from people they follow
- User should be able to page through their feed

***For the sake of this interview/ or for all the interview we can assume that the user is already authenticated and we have their UserId stored in session or jwt***


**Non Functional**:
- System should be **highly available**,(availability > consistency) eventual consistency with a 
- Posting and viewing the feed should be **fast < 500ms**
- Should be able to **scale**/handle 2B users
- User should be able to **follow unlimited** number of users and User Should be allowed to have **unlimited number of followers**.

## Design Constraint
Hard part of the problem is handling users having huge following/ users that follow huge number of people.

## Core-entities
**User**: User in our system.
**Follow**: A unidirectional relationship between users in the system.
**Post**: A post made by a user in our system, other users following the Owner of the post(another User) should be able to see this post in their feed. 

## Api or interface
Api's are the primary interface using which the user will interact with the system, just define each of your endpoints for each of our functional requirements.

`New post`

```json
POST /post/create
Request:{
    "content":{

    }
}
Response:{
    "postId"://...
}
```
Leave the content blank to account for more reach content or structured data that we might want to add in the post 

`Follow a user`
```json
POST /user/[id]/follow
Request:{

}
```
Follow operation will be binary, we will assume it to be idempotent, so it does not fail even if the user clicks on the follow button twice.

`View feed`

```json
GET /feed
Response:{

}

```
## General flow Optional
## High level design

- **User Should be able to post** and access it via postId
  -   We are going to build a basic flow and add more complexities later on
  -   Since we know we will have to scale later we will put horizontally scaled service behind the **API gateway/load-balancer**
  -   Leaving caching for the deep dive part
  -   We can handle more traffic by scaling horizontal i.e by  addition post-service
  -   User hit API gateway, which sends request to one of the instances of post-service which creates an insert event in db.
  
  ![post](image.png)

  -   For **database** we can use key-value store like **Amazon's DynamoDb** for the sake of its simplicity and scalability and we can provision nearly limitless storage provided we spread our load evenly on our partitions
-   **User should be able to follow** other Users
    -   
-   **User should be able to view a feed of posts** from people they follow




## Deep dive

## Tips

## Questions
    What is api gateway? is it any different from load balancers ?