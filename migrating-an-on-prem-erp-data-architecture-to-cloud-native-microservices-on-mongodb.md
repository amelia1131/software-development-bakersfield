# Migrating an On-Prem ERP Data Architecture to Cloud-Native Microservices on MongoDB

In my role as the chief architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most challenging endeavors I've undertaken involved the transformation of an outdated monolithic ERP system into a modern microservices architecture. This legacy monolith had become a costly burden to maintain, hindering my client's ability to innovate and adapt.

Fortunately, I've encountered similar challenges through years of delivering custom [Software Development Services in Bakersfield, CA](https://hybridwebagency.com/bakersfield-ca/best-software-development-company/). I understand the difficulties organizations face when they're weighed down by inflexible, challenging-to-modify systems. In this article, I'm excited to share our step-by-step approach to revamp the data model and decompose the monolith at both the code and database levels.

Throughout this transition, my team and I uncovered valuable, lesser-known best practices for modeling domain entities in a document database like MongoDB to support fully independent microservices. If you've ever wondered how to modernize your database architecture for the cloud while preserving historical data, this article will provide you with insights you won't want to miss.

By the end of this article, you'll have a comprehensive blueprint for migrating your own legacy systems to a modern, flexible architecture. I'll share practical insights to help you navigate common challenges and speed up the delivery of new features to your customers. Let's get started!

## Advantages of a Microservices Architecture

The adoption of a microservices architecture offers numerous benefits over the monolithic approach. It allows for the independent deployment of services, facilitating the rapid development and release of new features without disrupting the entire application.

Additionally, microservices enable the use of different programming languages and frameworks for each service, ensuring that the best technologies can be chosen for specific domains. For example, a recommendation engine might leverage Python's machine learning libraries, while a user interface can be developed using React.

This decoupling of responsibilities empowers specialized teams to work autonomously on individual services. It allows for swift validation of ideas through prototypes, reducing the risk associated with a full monolithic rewrite. New team members can quickly become productive by contributing to a specific service that matches their skills.

In an ever-changing technological landscape, microservices mitigate the risks associated with evolving technologies. Replacements only affect small, isolated parts of the system, rather than requiring a complete overhaul. Clearly defined interfaces make the migration of components to new implementations a seamless process.

#### Independent Scaling

Effective resource management is achieved when services can scale independently in response to demand. For example, during holidays or special events, only the order processing service might need additional servers to handle the surge in orders without affecting the entire system.

```
ordersAPI.scale(replicas: 5);
```

Horizontal scaling at the service level results in significant cost savings by efficiently allocating resources. Inactive support microservices, like user profiles, don't require costly overprovisioning to handle traffic that doesn't impact them.

## Analyzing the Data Model
### Understanding Entity Relationships

The first step in transitioning to a microservices architecture is to analyze how entities are interconnected within the existing monolithic data model. We thoroughly examined each collection in the MongoDB database to identify areas where domains naturally cluster and transaction boundaries are established.

Entities such as Users, Products, and Orders served as the core of these bounded contexts. The relationships between these fundamental entities became candidates for service decomposition. For instance, we observed that Orders contained foreign keys linking them to Users (representing customers) and Products (representing purchased items).

To gain a deeper understanding of these cross-dependencies, we visually mapped interconnected fields by examining sample documents. This process revealed that legacy code had merged data relevant to different business capabilities, such as shipping addresses repeating user profiles instead of maintaining lightweight references.

Our analysis of these relationships exposed instances of problematic tight coupling between modules, causing cascading updates. Normalizing redundant data eliminated obstacles to the independent development of user profiles and shipping namespaces.

We employed database tools to explore these connections. Using MongoDB Compass, we diagrammed relationships through $lookup pipelines and ran aggregate queries to count references between entities. This approach helped us identify key breakpoints for separating logic into coherent services.

Understanding these relationships not only delineated domain boundaries but also ensured that services offered well-defined interfaces. These clear contracts made it possible for autonomous teams to develop and deploy modules incrementally, akin to micro Frontends, without hindering each other.

### Identifying Transactional Boundaries

In addition to understanding relationships, we delved into transactions within the existing codebase to comprehend how business processes flowed. This exercise helped us identify areas where data modifications needed to occur within a single service to maintain data consistency and integrity.

For example, within the realm of order processing, we recognized that any updates related to the order itself, payments, inventory levels, and shipment notifications needed to take place within a single service to ensure data consistency. This insight played a pivotal role in defining the boundary of our Order Management service.

Analyzing both relationships and transactions provided crucial insights that guided the restructuring of the data model and logic into independently deployable microservices with well-defined interfaces.

## Refactoring for Microservices

### Standardizing Data Schemas

To support independent services that may deploy to different data stores when necessary, we standardized schemas to eliminate redundancy and include only the essential data required by each service.

For instance, the original Orders schema contained the entire User object. We carried out a significant refactoring to transform it into a lightweight reference:

```javascript
// Before
Orders: {
  user: {
    name: 'John',
    address: '123 Main St...' 
  }
  //...
}

// After  
Orders: {
  userId: 1234
  //...  
}
```

In a similar vein, we extracted product details from the Orders collection and assigned them to their independent collections, allowing these entities to evolve independently over time.

### Embracing Domain-Driven Design

We harnessed the power of bounded contexts from Domain-Driven Design to logically segregate services, such as Order Fulfillment and User Profiles. These interfaces abstracted data access:

```javascript
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

To align with the demands of the new architecture, queries and commands underwent substantial refactoring. Previously, services accessed data directly using calls like `db.collection.find()`. We introduced data access libraries to introduce an abstraction layer:

```javascript
// order.service.ts

constructor(private ordersRepo: OrderRepository) {}

getOrders() {
  return this ordersRepo.find();
}

// mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This approach granted us the flexibility to migrate databases without necessitating alterations to consumer code.

## Deploying Microservices

### Independent Scaling

In the realm of microservices, autoscaling is executed at the individual service level rather than on an application-wide basis. We implemented scaling logic through Docker

 Swarm and Kubernetes.

Our deployment manifests defined scaling policies based on CPU and memory usage:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]  
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

This scaling approach provided reserved capacity and efficiently managed elastic overflow. The Orders service autonomously monitored its performance and initiated or terminated containers as required:

```javascript
# orders.js

const cpusUsed = process.cpuUsage();

if(cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if(cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers, such as Nginx and Traefik, efficiently routed traffic to scaled replica sets. This approach optimized resource utilization, enhanced throughput, reduced latency, and effectively cut costs.

### Enhancing Resilience

Our journey also involved implementing a range of resilience techniques, encompassing retry policies, timeouts, and circuit breakers. Rate limiting and throttling were implemented to guard against cascading failures. The Platform service took charge of transient issues related to dependent services.

To bolster our resilience, we employed a combination of in-house solutions and open-source libraries, including Polly, Hystrix, and Resilience4j. Centralized logging via Elasticsearch allowed us to trace errors across the distributed applications.

### Strengthening Reliability

To ensure the reliability of our microservices, we deployed various techniques aimed at preventing single points of failure. Our primary focus was on automating responses to transient errors and enhancing protection against overload.

We leveraged the Resilience4J library to introduce circuit breakers for graceful fault handling:

```python
# OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting was employed to prevent services from becoming overwhelmed during periods of high stress:

```python
# RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts were introduced to terminate long-running calls:

```python
# ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

Retry logic was enforced via policies defined at the client level:

```python
# RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy was introduced to provide a more reliable experience, and policies were defined at the client level:
    policy.setMaxAttempts(3);
    return a RetryTemplate(policy);
  }

} 
```

These measures ensured uniform responses and prevented cascading failures across services.

## In Conclusion

The journey of transforming our monolithic ERP system into a microservices architecture has been an immense learning experience. This endeavor transcends the realm of technical migration, signifying a profound organizational transformation that empowers our client to serve the evolving needs of their customers more effectively.

By dismantling tightly interwoven layers and establishing distinct boundaries between domains, our development team has unlocked a new realm of agility. Features can now be developed and deployed independently based on business priorities rather than being constrained by architectural limitations. This newfound ability to experiment and iterate swiftly will keep our application ahead of evolving market demands.

In parallel, our operations team now boasts complete visibility and control over each system component. Anomalous behavior can be detected early through enhanced monitoring of individual services. Scaling and failovers have transitioned from manual to automated processes. This heightened resilience serves as a reliable foundation for our client's continued growth.

While the benefits of microservices are evident, embarking on such a migration is far from straightforward. Our approach to analyze relationships, define interfaces, and introduce abstractions, rather than opting for a crude 'rip and replace' strategy, has resulted in a flexible architecture capable of evolving in line with customer needs.

Above all, I am grateful for the opportunity this project provided to work closely with a client on their digital transformation. Sharing our experiences and insights here is a way to pay it forward, enabling others to learn from our successes and mistakes. My hope is that it empowers more businesses to embark on the journey of modernization, with the ultimate rewards being enhanced customer experiences.

## References 

- MongoDB Documentation - Official documentation on data modeling, queries, deployment, and more: [Read more](https://docs.mongodb.com/)

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles: [Read more](https://martinfowler.com/articles/microservices.html)

- Domain-Driven Design - Eric Evans' book introducing DDD concepts for structuring services around business domains: [Read more](https://domainlanguage.com/ddd/)

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps readily deployable to the cloud: [Read more](https://12factor.net/)

- Container Journal - In-depth articles on containerization, orchestration, cloud platforms: [Read more](https://containerjournal.com/)

- Docker Documentation - Comprehensive guides for building, deploying, and managing containerized apps: [Read more](https://docs.docker.com/)

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture: [Read more](https://kubernetes.io/docs/home/)
