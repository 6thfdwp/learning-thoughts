## Software Architecture

A description of the software architecture, including static structure (e.g. containers and components) and dynamic/runtime behaviour.


### Application Layer
Components in this layer are mostly handlers to be invoked when certain things are triggered, e.g upon receiving requests, time is up to triggered background jobs in the worker service, Lambda receiving events from the queue etc.

#### Bootstrap api service 
Inside the api service, all routers are defined in `pkg/api`, and bootstrapped from `cmd/api/main.go`
```mermaid
classDiagram
    class MainRouter {
        basePath = "/api"
        +Mount(subRouters)
    }

    class NudgeRouter {
        pathPrefix = "/nudges"
        +void Mount(*gin.RouterGroup)
    }
    class CopilotRouter {
        pathPrefix = "/copilotnudge"
        +void Mount(*gin.RouterGroup)
    }

    class NudgeAnalyticsRouter {
        pathPrefix = "/conversations"
        +void Mount(base *gin.RouterGroup)
    }
    class NudgeCommsRouter {
        pathPrefix = "/communications"
        +void Mount(*gin.RouterGroup)
    }
    class XXX {
        pathPrefix = "/..."
        +void Mount(*gin.RouterGroup)
    }

    class NudgeService {
       <<interface>>
       +PersistNudgeCampaign()
    }

    MainRouter --> NudgeRouter : Mount
    MainRouter --> CopilotRouter : Mount
    MainRouter --> NudgeAnalyticsRouter : Mount
    MainRouter --> NudgeCommsRouter : Mount
    MainRouter --> XXX : Mount

    NudgeRouter --> NudgeService : Persist NudgeEntity
    CopilotRouter --> NudgeService : Persist NudgeEntity
```
As you see when it comes to creating a new nudge, it will always depend on NudgeService interface to persist. Other routers provide endpoints to serve dashboard etc, which can directly depend on certain Repo interface if logic is simple to just query db result.

#### Bootstrap worker service 
Similarly, worker service instantiates all job runners (subworkers) are defined `pkg/workers`, bootstrapped from `cmd/worker/main.go`.  

```mermaid
classDiagram
    class MainWorker {
        +Start(subWorkers)
    }

    namespace nudge producers {
      class GigLifecyclesWorker {
         +Start(s *gocron.Scheduler)
      }
      class CopilotAwarenessWorker {
         +Start(s *gocron.Scheduler)
      }

      class AutoNudgeWorker {
         +Start(s *gocron.Scheduler)
      }
    }
    class ScheduledNudgeProcessor {
      +Start(s *gocron.Scheduler)
    }
    
    namespace hooks {
      class NewsletterWorker {
         +Start(s *gocron.Scheduler)
      }
      class JobOwnerWorker {
         +Start(s *gocron.Scheduler)
      }
    }

    class NudgeService {
       <<interface>>
       +PersistNudgeCampaign()
       +FindDueSend()
    }
    class NudgeCommsService {
       <<interface>>
       +Persist()
    }
    class SMSServiceAdapter {
         <<interface>>
         +Send(Notification)
    }

    MainWorker --> GigLifecyclesWorker : Start
    MainWorker --> CopilotAwarenessWorker : Start
    MainWorker --> AutoNudgeWorker : Start
    MainWorker --> ScheduledNudgeProcessor : Start
    MainWorker --> NewsletterWorker : Start
    MainWorker --> JobOwnerWorker : Start

    ScheduledNudgeProcessor --> NudgeService : Find unsent nudges and contactable recipients 
    ScheduledNudgeProcessor --> NudgeCommsService : Persist NudgeCommsEntity (per contact)
    ScheduledNudgeProcessor --> SMSServiceAdapter
    GigLifecyclesWorker --> NudgeService : Persist
    CopilotAwarenessWorker --> NudgeService : Persist
```
- `SheduledNudgeProcessor`  
   It is a special one running in the same worker instance. It processes all unsent nduge entities produced from different sources, could be from api requests, or other sub workers. It mainly depends on both `NudgeService` to find, and store contact communications via `NudgeCommsService` methods.  

- Nudge Producers  
Other workers like `CopilotAwareness` or `GigLifecycleWorker` are considered nudge producers, it depends on same `NudgeService` to create and persist nudges in db, to be picked up by `ScheduledNudgeProcessor`

- Hooks  
  These are sent to HR users as described in [Functional Overview](02-Functional-Overview.md). All emails are sent from these workers directly without async processing.


#### Lambdas
Lambda will mutate nudge converstaions and customer's contact_commnications tables upon receiving any events related to emails through nudge apis. So Lambda is not aware of unerlying data schema and mutation logic, and no need to maintain same db connections as nudge api service already has the connection knowledge.

- `cmd/consumer/main.go`   
This creates `lambdas.OutgoingEmailHandler`. It consumes outgong sqs and call Pinpoint for each notifiaction event one by one. Update contact_communications with ref_id from Pinpoint (used to link reply, see below). The nudge api involved: `/api/communications/%commid`
- `cmd/analytics/main.go`  
This creates `lambdas.AnalyticsHandler`. It consumes analytics sqs and update nudge conversations and store event in the contact_communication_events. The nudge api: `api/conversations/:id/analytics`
- cmd/email_receiver/main.go  
This creates `lambdas.IncomingEmailHandler`. It will parse email content, extract reference id out of it and store attached files in S3 (if exists). Then it persists one reply communication record against original communication by looking up its reference id. The nudge api: `api/communications/reply`



### Domain Layer
They are defined in `internal/domain`, all the domain entities/DTO objects, services should be here.   
Currently we've implemented core logic related to nudge campaigns and manage their related communications, which can be reused to serve nudge categories triggered from differnt sources. We can gradually flesh out domain services if we identify more use cases.

The layer abstracts complext domain logic from application. Service methods serve aggregate roots to deal with `NudgeEntity`/`NudgeCommsEntity`, as they are always treated as a whole from business perspective:  

NudgeEntity always composes 1 nudge record linked with 1 campaign record. Same as NudgeCommsEntity composes 1 converstaion record (in nudge db) and 1 contact communication record in customer db.

```mermaid
classDiagram
    class NudgeService {
        +FindDueToSend() []NudgeEntity, error
        +FindRecipients(NudgeEntity) Contacts, error

        +PersistNudgeCampaign(NudgeEntity) int64, error
    }

    class NudgeCommsService {
        +Persist(entities.NudgeCommsEntity)
    }

    class NudgeRepo {
       <<interface>>
       +Create()
       +CreateCampaign()
    }
    class ContactRepo {
       <<interface>>
       +FindByIds()
       +FindByJobId()
       +FindByTalentProjectId(NudgeCommsEntity)
    }
    class ConvAnalyticsRepo {
       <<interface>>
       +Create(NudgeCommsEntity)
       +UpdateAnalytics(NudgeCommsEntity)
    }
    class ContactCommsRepo {
       <<interface>>
       +Create(NudgeCommsEntity)
       +UpdateMessageId()
       +CreateEvent()
    }

    namespace entities {
      class NudgeEntity {
         +string  Type
         +[]string Channels
         +[]int64  ContactIds
         +Campaign // subject, content etc
         +SetContactableIds()
         +SetTemplate()
         +SetSchedule()
      }
      class NudgeCommsEntity {
         +int  NudgeId
         +bool Delivered
         +bool Opened
         +bool Clicked
         +[]int64  ContactIds
         +ContactComms // subject, Html/TextContent etc
         +SetRichContent()
         +ToNotificationEvent()
      }
    }

    NudgeService --> NudgeRepo : Persist 
    NudgeService --> ContactRepo : FindRecipients
    NudgeService --> NudgeEntity : Use 
    NudgeCommsService --> ConvAnalyticsRepo : Persist 
    NudgeCommsService --> ContactCommsRepo : Persist 
    NudgeCommsService --> NudgeCommsEntity : Use 
```

### External Infra (Adaptor/Repository)
Repository can be seen as one type of Adaptors that specifically deal with external data access. They are located in `repos/{subfolder}` packages to be organised around data concerns. The implementations are mainly db related as we access data directly from db. 
```mermaid
classDiagram
    class NudgeRepo {
       <<interface>>
       +Create(NudgeEntity)
       +CreateCampaign(NudgeEntity)
    }
    class ConvAnalyticsRepo {
       <<interface>>
       +Create(NudgeCommsEntity)
       +UpdateAnalytics(NudgeCommsEntity)
    }
    class ContactCommsRepo {
       <<interface>>
       +Create(NudgeCommsEntity)
       +UpdateMessageId()
       +CreateEvent()
    }

    class nudgeDBRepo {

    }
    class convAnalyticsDBRepo {
      
    }
    class contactCommsDBRepo {

    }
    class DBSelector {
      <<interface>>
      +GetNudgeDB()
      +GetCustomerDB(customer)
    }

    NudgeRepo <|.. nudgeDBRepo : implements
    ConvAnalyticsRepo <|.. convAnalyticsDBRepo : implements
    ContactCommsRepo <|.. contactCommsDBRepo : implements
    nudgeDBRepo --> DBSelector
    convAnalyticsDBRepo --> DBSelector
    contactCommsDBRepo --> DBSelector
```
NOTE: These in the diagram above are some core data access logic related to creating and processing nudges. We haven't migrated all repos to this structure. They exist under `pkg/repos` folder as individual source files. The implementation lacks proper abstraction, is coupled with concrete db types, more like table wrappers. We should plan to review them to follow the consistent structure as above. 

Similarly current external infra logic are implemented under `pkg/clients`, e.g pinpoint, sqs. Again they lack proper abstraction and did not conform to [Principles](05-Principles.md).  

We've started adapter implementation for new SMS communication. They are located in `internal/adapters`. We can split to sub packages based on external concerns.  
```mermaid
classDiagram
   namespace comms {
      class SMSServiceAdapter {
         <<interface>>
         +Send(Notification)
      }

      class smsMtnAdapter {
      }
      class twilioAdapter {
      }

      class EmailServiceAdapter {
         <<interface>>
         +Send(Notification)
      }
      class pinpointAdapter {

      }
      class xxxAdapter {

      }      
   }

   SMSServiceAdapter <|.. smsMtnAdapter
   SMSServiceAdapter <|.. twilioAdapter
   EmailServiceAdapter <|.. pinpointAdapter
   EmailServiceAdapter <|.. xxxAdapter
```
By following the principles, our interface exposed to other layers are infra agnostic. e.g `Notification` is DTO objects from domain layer. So it'd be easy to swap another provider to send nudges without impacting other parts. 

### Audience
Technical people in the development team.