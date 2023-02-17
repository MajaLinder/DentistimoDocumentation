# Dentistimo
This is the main repository documenting the Dentistimo software developed during seven weeks by Team 4 as part of the course DIT355 Mini Project: Distributed Systems Development given at the University of Gothenberg. Dentistimo is a distributed system for booking dentist appointments, and has been developed by Team 4, a team of seven highly interested software engineering students using a Scrum process. The main focus of the project has been to get an introductory experience developing a distributed software architecture, and has been the developers' first exposure to the application of software architecture concepts in practice implementing a distributed system.

## Project Description 

Dentistimo offers users a quick, reliable, and simple-to-use service for booking appointments at dentistry clinics in the city of Gothenburg. Users can quickly locate a dentistry on an interactive map of the city, view the available timeslots for that particular dentistry as well as place a booking. The simplicity of use allows for a positive user experience and the service eliminates the need to contact various different dentistries in order to get a dentist appointment. With Dentistimo, it is now quick and easy to find an appointment at a dentistry near you that fits your schedule. 

## Overview: Components and Technologies Used
The system has been developed using a variety technologies for software development, most importantly the web languages (HTML, CSS & JavaScript), as well as technologies specific to front end (Bootstrap  & Vue.js), back end (MongoDB & Node.js) as well as using the MQTT messaging transfer protocol for communication between components. The system is distributed over four different components, each with its own responsibility supporting the overall use case of the system as a whole. The individual components are described in detail in their respective repositories.

1. [ClinicRegistry](https://git.chalmers.se/courses/dit355/test-teams-formation/team-4/clinicregistry)
2. [UI](https://git.chalmers.se/courses/dit355/test-teams-formation/team-4/ui) 
3. [TimeSlotGenerator](https://git.chalmers.se/courses/dit355/test-teams-formation/team-4/availabilitychecker)
4. [AppointmentHandler](https://git.chalmers.se/courses/dit355/test-teams-formation/team-4/appointmenthandler)

For testing the fault tolerance of the system, an additional Testing component has also been developed, which doesn't interfere in the overall functionality. The repository for the Testing component can be found [here](https://git.chalmers.se/courses/dit355/test-teams-formation/team-4/tester).

Additionally, the system is connected to a NoSQL database that has been implemented in MongoDB to store the appointment bookings made in the system. 

![Software component diagram](/diagrams/Component_diagram.png)

## Software Architecture 

![Software Architecture](/diagrams/Architecture.drawio.png)

### Forces and drivers

The system has been developed with particular consideration to three specific quality attributes derived from [the assignment briefing document](/assignment_briefing_dit355.pdf) provided by the course teacher at the onset of the course. These quality attributes were thus translated into Architectural Significant Requirements for [the first sprint retrospective](/annotated_retrospective_sprint_1.pdf) where they were stated as follows:

#### Scalability

> - The system shall be **scalable** as many users will use the system at the same time.
We believe that the system shall give our users the ability to be used through a
desktop computer, iPad or a mobile phone for example.

Scalability has been very prominent in the architecural decisions taken and design choices made on all levels throughout the development process. The way that the delivered system supports scalability is two-fold. 
- Firstly, by having distinct and clearly delimited single areas of responsibility for each component scalability is supported in terms of scaling functionality. Let's say that in the future the product owner comes with new requirements regarding the canceling of booked appointments, or maybe a new user story concerned about removing certain available timeslots during the holidays, then the architecture supports scalability in that the componentization of different areas of responsilibilty makes it easy to add further functionality later on, as well as determining in which components the corresponding new code will likely belong.
- Secondly, and most importantly for the scalability quality attribute, is for the system to be able to handle a growing number of simultaneous users, especially taking into account concurrent booking requests made by different clients. The system does this by attaching a special issuance attribute to booking requests, which get placed in a priority queue in the AppointmentHandler component. The issuance is derived from the JavaScript Date object which returns a single moment in time as measured in milliseconds since 1 January 1970 UTC, and will differentiate booking requests made simultaneously, allowing only one (the one made first as measured in milliseconds) to be confirmed and the others rejected. Another design choice made with scalability in mind is that of reading the JSON file of dentistries continuously and publishing the data upon request from the UI component so to make it visible to the user. This makes the system able to handle larger files of dentistries seamlessly if the system is to scale simply by adhering to the same data format. 

#### Performance

> - The system shall respond within one second to users' search for clinics, time
booking, etc. Therefore **performance** will be of high priority.

- The overall complexity of the system resides mainly in two components, namely the AppointmentHandler and the TimeSlotGenerator. The reason for this is that only the booked appointments get stored in the database. The database doesn't store all the timeslots for all the clinics for some amount of time in the future. Instead, on a click from the client to check the availability of a specific clinic the available appointments for that specific clinic get generated and published to the UI. Therefore there might be a concern that the TimeSlotGenerator would decrease the system's performance. This issue has been taken into consideration as available timeslots always get generated for the coming seven days ahead (this is a random number chosen for the implementation and could be updated easily). Another issue that arose after the decision was taken of only storing the booked appointments in the database is that the number of dentists at each clinic would need to be considered and how the same timeslot would need to be bookable the same number of times as there dentists in that clinic. This issue was solved by using a data structure to calculate how many appointments with a particular timeslot has already been stored in the database.
- In the AppointmentHandler component the booked appointments get stored in the database after the component has checked if the appointment is bookable or not. Here, the case might be that there will be a large number of calls for booking appointments. They might also happen simultanuously. In addition the network latency and the potential network overload has been kept in mind in the solution implemented for these issues. A decision was made to implement a priority queue where bookings get handled based on a timestamp that gets issued when a booking gets requested in the UI. A booking gets handled every 1 millisecond where the client who first requested the timeslot gets the appointment if it's bookable. 

#### Availability

> - The system needs to be **available** so the users can search for clinics at any time to
be able to book an appointment.

- Availability has been at the core focus of the system which relies in the application's independent components communicating to each other for completing a task, via the middleware, meaning that components will continue to function even if one or more are down. This in turn ensures that the service will be able to deliver gracefully. 
- Regarding fault taulerance, a circuit breaker pattern has been implemented using Opossum, which is implemented into the TimeSlotGenerator component, as the most central component. The circuit breaker is triggered in the occasion that the requests take longer than 3000ms, or when more than 50% of them fail. It resets to try again after another 30 seconds. Thus it provides a means for the system to recover and resume its function. 

### Architecture styles

#### Client-server architecture style

Since Dentistimo is a web-based software it has been developed with a client-server architecture, since that is the architecture style inherent to the web architecture itself. The UI component is the client in this architecture, utilizing data retrieved from the other three components to interface with the user, and displays that data visually in different ways to make for a good user experience. It also lets the user input data to be sent to other components e.g. to make a booking using an HTML form element. 

The ClinicRegistry, AppointmentHandler and the TimeSlotGenerator components are all servers in this architecture, and they all interact in the data flow directly with the UI as well as some of them communicating directly with each other as well. The AppointmentHandler and TimeSlotGenerator also have database connections, to which they either read or update data depending on the use case. 

- The ClinicRegistry for example serves the client by fetching data from a URL which then get displayed on a map interface, implemented using the online map provider MapBox in the client. 

- The TimeSlotGenerator component makes the client display the available timeslots for any given dentistry based on client input. It reads the database to be able to make timeslots that have already been booked not be displayed, and it returns the available timeslots to the client in a list, which gets continuously updated and re-published as inter-server communication with the AppointmentHandler happens every time a new booking is made and saved to the database.

- The AppointmentHandler validates bookings made in the client as well as updates the MongoDB database when a booking is placed and then makes a confirmation or rejection being displayed in the client. Whenever a booking has been saved to the database, this server lets the TimeSlotGenerator know it needs to regenerate a new list of available timeslots to be shown in the client.

#### Publish-subscribe architecture style

All data communication happens via the MQTT publish-subscribe network protocol, implemented using the JavaScript mqtt.js library in all components. The use of MQTT was a requirement from the [assignment briefing document](/assignment_briefing_dit355.pdf) for the project, and influences the choice of architecture styles as well. The publish-subscribe architecure style is an event-driven architecture style, which means that network communication is event-based. When a component wants to communicate with another it simply publishes a topic containing a message for that other component, which then has to subscribe to that particular topic in order to receive the message. This style is very suitable for distributed systems, as it naturally supports network transparency and low coupling between components. The publisher doesn't know anything about the subscriber and vice versa, and the architecture style supports scalability, so that if additional components are to be added and require certain data, all they will need to do to successfully interface with the existing system is to subscribe to the same topics via the same broker. 

#### Layered architecture style

The system is also implemented in a layered architecture style, in the sense that each component has been associated with a particular layer, corresponding to the component type. The layers in question for the system are:
- the Presentation layer, corresponding to the client in the client-server architecture style,
- the Application layer, corresponding to the servers in the client-server architecture style, 
- the Data layer, corresponding to the data sources in the client-server architecture style,
- the Middle-ware layer, corresponding to the MQTT broker in the publish-subscribe architecture style.

The division into layers like this makes the division of responsibilities as well as the flow of data between layers more clear, which allows for further functionality to be added more easily and the software to be more readily expanded upon.

#### Quality of Service per component

AppointmentHandler:
The AppointmentHandler component publishes to three topics. 
- The topic /requestAppointment is about requesting data from the ClinicRegistry component to be able to get the number of dentists from the JSON URL. This has QoS (Quality of Service) 1 because it has to be sent at least once but can handle the same message being sent again. 
- As for /Confirmation, this topic has to have QoS 2 since the UI that receives the message would not be able to handle more than one message to the same client. 
- The last topic, /clinicData, publishes clinic information to the TimeSlotGenerator so that it can use the clinic name and opening hours to generate a new list of available time slots after a booking is made. It is fine if the message is sent more than once since the list can get generated again on request, therefore QoS is 1.  

UI:
The UI component publishes to three topics. 
- The topic /bookingRequest has QoS 2. It is important that the message is sent exactly once to the AppointmentHandler because otherwise a booking could get saved more than once. 
- The second topic, /requestList, requests clinic information from the ClinicRegistry and has QoS 1 since it has to be sent at least once. 
- The last topic, /clinicData, publishes clinic information to the TimeSlotGenerator so that it can use the clinic name and opening hours to generate a new list of available time slots. It is fine if the message is sent more than once since the list can get generated again on request, therefore QoS is 1.  

ClinicRegistry:
The ClinicRegistry component publishes to two topics. 
- With the first topic, /jsonData, a message is sent to the UI to display markers and clinic information on the map in the UI. It has QoS 1 because the map can handle receiving another message and update, but the message must be sent at least once. 
- As for the second topic, /jsonDataToAppointment, it has the same logic as described above with the topic /requestAppointment, and has therefore QoS 1. 

TimeSlotGenerator:
The TimeSlotGenerator component publishes to one topic. 
- The topic /availableTimeslots sends a message which is recieved by the UI. Available time slots can be handled multiple times in the UI if a message would be sent more than once, so QoS is 1.

### Additional design decisions

#### Database decisions

Two main strategies were identified regarding storing and retrieving data from the database where to (i) generate ahead of time available timeslots and store them in the database
and (ii) only store timeslots when they get booked by a user 

For the first strategy  we had to
-   Designate a table for available timeslots for all dentistries (eg 1 week ahead of time)
-   Every week a process should be run to generate the available timeslots ahead of time
Its advantages were that searching was quicker and easier (i.e. query the table and get the slots within given dates and times that are not taken),

Its disadvantages however were more, including: 
-  There needed to be a functionality that is called periodically to pre-generate available slots for the coming period
-  The table that would hold the available slots could have gotten quite big with time which means that reading speed would have suffered (qurying might get slower and slower). So some mitigation strategy needed to be in place in order to have a performant and scalable solution.
-  If we were to read another registry of dentists with different number of dentists and/or a new length for a timeslot, it would mean that the available slots already generated need to be regenerated while taking care of the already booked ones. 

For the second strategy, the advantages were
-   Small table to hold the data for timeslots that are booked
-   Query for getting the booked timeslots within given interval was also simple
Its disadvantage was that the calendar of available timeslots for the given period needs to be generated during run-time and there has to be some logic to block the already booked timeslots. 
After careful consideration of these strategies we decided to implement the second one, we store on our database appointments when they are booked, complying with our architectural drivers scalability and performance. 

Since all booked appointments are saved in the database and the TimeSlotGenerator generates new time slots based on them, a discussion came up regarding already passed appointments and performance. In the component TimeSlotGenerator, all booked appointments get retrieved from the database and it is excluding those that have passed the current date. This means that after a while, the database will store a lot of already passed appointments and the TimeSlotGenerator would have to go through and exclude all of them which would decrease the performance. A suggestion came up to delete already passed appointments in the database and assume that these would be archived somewhere else. However, the final decision was to not pursue this suggestion because of scalability and traceability, to be able to view and use the history of appointments if desired by the customer and then added to the requirements later. For the future we would have to implement an archive method following good industry practices and legal requirenemnts. 

#### Placement of booking validation methods

##### Tradeoff and discussion

The team had a big discussion about the tradeoffs of putting the methods for validating a booking request for dentistries with more than one dentist in the AppointmentHandler or the AvailabilityChecker. Fundamentally the issue had to do with single responsibility when we identified five responsilibilties and tried to distribute them over four components. One group of developers saw the fifth responsilibilty (validating booking requests) as being a part of the third responsibility (appointment handling) and therefore belong to the third component (the AppointmentHandler), while another group of developers saw the responsibility as part of the fourth responsibility (checking for availability) and therefore belonging to that one (the AvailabilityChecker). An overview of the proposed solutions can be found [here](https://drive.google.com/drive/u/1/folders/1zGXFXyeBI_EZFLGHaylXg5I4fkdvzdKt). When we finally agreed that we were talking about a new responsibility area we needed to make the tradeoff as a team of which component to give a double responsibility, since we did not want to create an entirely new component so late in the development process. Both groups of developers favoured putting the new-found responsibility in the component themselves had worked on previously. After some discussion the team compromised and chose to put the booking request validation in the AppointmentHandler and to rename the AvailabilityChecker to TimeSlotGenerator for the responsibility to be more transparent. This insertion of new functionality onto the AppointmentHandler thus necessitated one extra network call, decreasing the expected performance. However, this might not have been so bad after all since the extra network call would have had to happen anyway in case we would have implemented the booking request validation in its own component. 

## ER Diagram

![ER Diagram](/diagrams/ERdiagram.png)

## Use Case Diagram

![Use Case Diagram](/diagrams/Use_Case_Diagram.drawio.png)

## Final Project retrospective

### Project management

The software has been developed by a team of seven people over eight weeks. The team has followed a Scrum process complete with four two-week sprints, a product backlog with requirements in user story format and daily Scrum meetings on top of additional sprint planning, review and retrospective meetings. Each sprint has commenced on a Wednesday and lasted two weeks. The sprints have all commenced with a sprint planning meeting in which the scope has been decided on for the sprint, based on estimation of the product backlog user stories that was done in the first and second sprint planning meetings. The Scrum master has hosted 15-minute daily Scrum standups where every team member has been expected to attend and report on their delivered work from yesterday as well as planned work for the day. The sprint has then ended with a review meeting with the team's assigned T.A. who has taken the role of software consultant, followed by a team retrospective where the sprint has been evaluated by the team members, in order for improvement to be continuously made during the course of the project. 

This process has been documented on a [Kanban board](https://trello.com/b/9RZSdBkE/dit-355-2021-team-4) using Trello, which has been used for planning as well as for documentation and tracking progress. Team members have self-assigned themselves to user stories and tasks in order to have transparency of the state of the project at all times. The Scrum master has used the Trello board as a framework for the daily standups and the board contains a complete log over the continuous progress made on the project by all team memebers across time.

The product backlog initially contained eleven user stories, written by the product owner in close discussion with the entire team, of which only four made part of the final delivery and the rest were deemed to be out of scope already by the second sprint. The user stories ended up being too large, not constituting a manageable size and scope for a single developer to deliver within a reasonable amount of time, and were therefore broken down into pieces, that have been referred to as both sub-user stories and tasks trhoughout the project. These tasks had the number of the user story, for example User Story 8, followed by a dot and the number of the sub-user story, for example 8.5, as their identifier, and one user story was not done until all of its constituent sub-user story parts were delivered, making up the larger user story. 

### Strategy and scheduling

The first half of the project turned out to be a phase that can only be described accurately as Big Design Up Front and consisted mostly of project planning. No team member had any previous experience to draw from regarding the implementation of a distributed system, and this project was also the first experience for every team member of actually applying software architecture concepts in practice. The team being cross-functional and eager to learn, the first couple of weeks saw a lot of discussion and debates about software architecture, drawing and redrawing architecture sketches and diagrams, and learning about the tradeoffs of going with different architectures for the final solution. A unified architecture consensus and implementation of that architecture didn't commence until about halfway through the second sprint. This BDUF did in fact turn out to be very fruitful, as there was very little in terms of disputes or conflicts as there seemed to have emerged a shared understanding from these lengthy discussions early on. These early architecture discussions also helped to strengthen and solidify everyone's understanding of software architecture theory, which of course helped the whole team later on as in the later sprints those concepts had to be applied in practice. 

The four user stories ended up being scheduled for delivery at a rate of about one per week, spread across all four components and seven developers. That may say something about the trouble the team had with breaking the user stories down properly but the schedule of delivery was managed to be met for two of the four user stories in the scope. For the remaining two, problems revealed themselves late in the last sprint as unforeseen requirements needed to be implemented as part of fulfilling the acceptance criteria of unfinished user stories, thus delaying their delivery from the planned delivery dates. As to measures that could have been taken to prevent this, a less waterfallish approach could have been one way to have managed time more effectivly, although the general idea among the team has been that the initial planning and architecture phase in the two first sprints was necessary in order to deepen everyone's knowledge and also to get to an architecture and design consensus that in fact might have benefited the work later on, so the team is unsure if this would have been a valuable measure to counter the delayed delivery from the planned one.

### Lessons learned and conclusion

#### Challenges

1. Managing effective code review
2. Effectively breaking down user stories


#### Solutions

1. The team used the Trello board to let all the members know when a user story or a task needed to get reviewed. However, towards the end of the project the team realized that the pull-based system of Kanban did not work as intended when it came to reviewing each other's work, as few team members assigned themselves to reviewing. This led to different understandings of how and where some functionalities and responsiblities should be implemented as sub-teams had different ideas and opinions. This could have been avoided with proper reviewing before confusion and long discussions arose. The team could have planned for another practice for reviewing code, such as making merge requests on GitLab and assign other members. This way the code would not have got merged in main until someone, in addition to the developers who created it, approved it. This would however not have solved all issues regarding different understanding and ideas, and some of the discussions were needed to get to a consensus and clear out any confusions. 

2. The way the team worked with breaking down user stories mentioned above could benefit of improvement since it made the progress on the entire user story hard to track and determine, and the team did not end up reviewing every finished task, which meant that a lot of code review was not done until the very last moment. One lesson derived here has been to take the time necessary to break down user stories effectively to begin with, in order to avoid a larger user story to be without any visible progress on the Kanban board for too long, thus negatively affecting the transparency of work.

#### Lessons learned

The project as a whole has been a very dense learning experience, and there have been lessons learned both in terms of project management, programming, as well as software architecture and effective teamwork. One of the most valuable lessons has been to experience first-hand the complexity of doing the architect's job as a team as everyone were involved in reasoning about, developing and implementing the software architecture. This led to more of a waterfall planning phase early on, typically characterized as Big Design Up Front, but which ultimately led to clarity and success in implementing what was to become the final software product. It was not until the last one-week sprint when the team was very close to finishing up the project that some inconsistencies in the shared vision made themselves prevalent, whose reason were some unforeseen requirements that had not been expressly stated but needed to be implemented in order to fulfill the acceptance criteria established and connect each others' work. Overall the implementation following the initial architecturing phase went smoothly, and this has given the team a renewed respect for BDUF with regards to software architecture, since it deals with the level of complexity that software architecture imposes on a project. BDUF gives a more detailed initial plan which might give more clarity across the whole team and raise motivation as it steers everyone towards the same goal. Similarly to the discussion that concluded the previous course on software architecture, we have now also had the first-hand experience that BDUF can be powerful in delivering evolutionary software architectures in combination with an agile workflow. 

### Individual contributions

**Bardha Ahmeti - Product owner**

Main responsabilities included identify, evaluate and develop architectural solution as well as documentation of project and software. 
Contribution in Components:  TimeSlotGenerator, AppointmentHandler, Testing and Documentation. User Stories: 8.2, 8.5, 10.1, Circuit Breaker. 

**Lina Boman - Developer**

Contribution in Components: AppointmentHandler, UI and Documentation. User Stories and tasks: 1.1, 1.2, 1.3, 1.4, 1.5, 3.1, 3.2, 3.3, 10.1, Frontend CSS styling, Setting up project template and Refactoring.

**Jina Dawood - Developer**

Contribution in Components: AppointmentHandler, UI, Clinic Registry and Documentation. User Stories and tasks: 1.1, 1.2, 1.3, 1.4, 1.5, 3.1, 3.2, 3.3, 10.1, Frontend CSS styling and Refactoring.

**Bassam Eldesouki - Developer**

Contribution in Components: TimeSlotGenerator, Appointment Handler, UI, Clinic Registry and Documentation. Tasks: Set up the project template, set up the MQTT client, refactoring, Fetch JSON from URL, Unsubscribe stopped components, User Stories: US 8 (8.3, 8.7, 8.5), US 10 (10.3, 10.2).

**Malte Götharsson - Developer**

Contribution in Components: AppointmentHandler, UI, ClinicRegistry and Documentation. User Stories and tasks: 1.1, 1.2, 1.3, 
1.4, 1.5, 3.1, 3.2, 3.3, Prototype, Bugfixes, Set up project template.

**Maja Linder - Developer**

Contribution in Components: TimeSlotGenerator, AppointmentHandler, UI, Clinic Registry and Documentation. User Stories and tasks: 3.3, 8.2, 8.5, 8.6, 8.8, 10.2, 10.3, Fetch JSON from URL, Update QoS, Error handling, Refactoring.

**Karl Stahre - Scrum master**

Main responsibilities included hosting daily Scrum standup meetings and enforcing adherence to Scrum and Kanban methodology and practice, as well as requirements and software documentation.
Contribution in components: Documentation, TimeSlotGenerator and AppointmentHandler. User stories: 8.1, 8.5, 8.8, 10.2
