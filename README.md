# SI Exam Project
## Made by
Adam Saidane as410  
Emil Valbak Hermansen eh108  
Sebastian Lundsgaard-Larsen sl281  


## Links to the project repositories
[Monolithic flight application](https://github.com/SOFTBoiS/SI_Exam_Monolithic_Flight_Application)  
[Eureka Server](https://github.com/SOFTBoiS/SI_MP_4_Eureka_Server)  
[Car Gateway](https://github.com/SOFTBoiS/SI_MP_4_Car_Gateway)  
[Car Catalog](https://github.com/SOFTBoiS/SI_MP_4_Car_Catalog)  
[Rent-A-Car](https://github.com/SOFTBoiS/SI_Exam_Rent_A_Car)  
[Translator](https://github.com/SOFTBoiS/SI_Exam_translator)  
[Review Catalog](https://github.com/SOFTBoiS/SI_MP_4_Review_Catalog)  
[Ad producer](https://github.com/SOFTBoiS/SI_exam_ad_producer)  
[Kafka metrics](https://github.com/SOFTBoiS/SI_exam_kafka_metrics)  
[Notification producer](https://github.com/SOFTBoiS/SI_Exam_Notification_Producer)  




## Introduction
We have developed a monolithic application and a variety of microservices for this exam project. This document describes our business case, the architecture of the application and its components.

### Business Case
The company we are developing our solution for has an existing monolithic application. Currently they have no way of making any form of targeted advertisement. They also want to add a new car rental service, to allow for booking of cars along with their flights. Furthermore, all of their business rules are hardcoded in the application, which means that any changes would require a new deployment of the application. The application is growing larger, feature by feature and it is slowly getting more complex to add new features. 

### System architecture
![](https://i.imgur.com/PdpIlVu.png)

The diagram shows our proposed solution, color coded with the development status of the components. In the microservice registry the boxes represents individual services. 
In the monolithic application the boxes represent features. The application uses a single relational database to store user, booking, flight and airport information. It contacts the Car-Gateway microservice with a HTTP request to get a list of cars, and later to create a car booking. Since it is a legacy system, it makes use of XML to serialize and deserialize objects, which is why the Car-Gateway contacts an XML/JSON transformer to translate between formats.




## Monolithic Flight Application
### Description
The Monolithic Flight Application is the main system. It is a legacy system which the company has had for many years. It has been used for creating flight bookings for users. We have expanded this application by connecting to microservices, which allows users to rent a car along with their flight booking.  
### Flow
The application is written in C# and built with ASP.NET and the architecture is as follows:  
A user is given a view. The view is either updated, or the user is returned a new view from the controller, after it has contacted a Model or the Car microservice. During the booking process the controllers contact the Camunda Rest-Engine to update the process instances with variables. If a flight or car booking should be cancelled, the camunda listener will handle the task of contacting the appropriate service.  
![](https://i.imgur.com/mKtQkdH.png)


### BPMN

We use BPMN and Camunda to automate cancellation of flights and cars. If the process is somehow interrupted, we have timers in Camunda that will make an external service call to cancel the flight booking, and car booking if one is made. Otherwise the user has the choice to cancel the booking at the end of the process if they change their mind. All of the external services are topic listerners in the Monolithic Flight Application  
![](https://raw.githubusercontent.com/SOFTBoiS/SI_Exam_Monolithic_Flight_Application/main/media/booking.png)

## Car rental service
Car rental service cosists of several microservices with Java Spring Boot with REST and a Eureka server. It gives the monolithic application the feature to add rental car bookings in addition to the flight bookings.
* The Eureka Server makes sure that the other microservices can connect to each other. 
* Car Gateway works as the communicator gateway for the other microservices. This is the the service that the monolithic application contacts to handle all requests. 
* The Car Catalog provides endpoints and sql database for storing and getting available cars for rental. 
* Rent-A-Car service provides a service for saving/deleting car bookings/orders in a mongodb database as well as functionality for getting orders by username or car id. 
* Car Review offer customers a opportunity to make reviews of the cars they have tried, and saves the reviews in MongoDB.
* Translator handles translation to/from json/xml.

Since these are microservices the Car Catalog, Rent-A-Car and Car Review microservices of course have their own individual databases. 

![](https://i.imgur.com/ecvuoxC.png)

### Load balancing
Estimating load balancing can be hard. Our initial thoughts are that since the three microservices relevant to cars (review catalog, rent-a-car and cat-catalog) all goes through the same gateway with synchronous calls, we will need a 3:1 ratio with the amount of gateways to individual car applications. Also to make sure we have High Availability, we should have more than one of each application. An example of this would be to have two of each of the car microservices, and 6 gateways to follow the ratio.

However, if car-gateway calls were async, it wouldn't have to wait on each response from the other applications and therefore the gateway would be less busy. Suddenly load balancing would be more focused on on the microservices related to car gateway. You would have to look at metrics about which one has the highest load, longer response times etc. since not all requests are equal and some endpoints will be contacted more often than others.

### Logging
We implemented logging functionality in the Java Spring Boot framework. By default it logs every level, but in production you should increase the logging level to reduce the amount of writes to the logging file. These logs are saved to the spring.log file in the root path. An example can be seen in the Car_Gateway microservice in the OrderSearchController in the POST "/orders/" endpoint. 

## Analytics
This microservice makes analytics of the kafka logs that contains car and flight search history. We think a good application to use this stream would be to use the log once a day, week or month to analyze that and save the aggregated data. This can be used for business intelligence, so you can make business and marketing decisions based on the aggregated data e.g. targeted ads. We use the stream as a sort of database of search history based on the given interval you want your aggregated data in.
![](https://i.imgur.com/7B9sUGH.png)




## Ad producer

The goal of this microservice is for a marketing team to send out marketing emails to a specific targeted audience. Example would be they've used our metrics analyzer microservice to figure out who they want to target and use this microservice to then send emails out.

With this microservice you hit an endpoint with your database query and the contents of the email. Application connects through gRPC (google RPC) to the monolithic flight application and retrieves all emails based on the query. The microservice then uses the mails to send out marketing emails with the specified content.

## Notification producer
When producing notifications we make use of RabbitMQ. Through an endpoint we can either send a notification about flight updates (schedule changes, delays etc.) to the respective phones' queue that are related to a certain flight number. This is right now done with a direct exchange. You can also send a general notification to everyone with a fanout exchange. Consuming the message would be done on the mobile application eg. with [React Native RabbitMQ plugin](https://www.npmjs.com/package/react-native-rabbitmq) or with [RabbitMQ's example](https://www.cloudamqp.com/blog/2015-07-29-rabbitmq-on-android.html) 
![](https://i.imgur.com/LIDj1dH.png)