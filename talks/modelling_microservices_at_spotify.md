# Modelling Microservices at Spotify

- Art of scalability

  - X axis: cloning - more copies of the same thing (horizontal scaling)
  - Y axis: splitting - microservices
  - Z axis: sharding - database data splitting

- Spotify (at 2016)
  - X: ~10k servers
  - Y: ~1100 things
  - Z: not much, using out-of-the-box approach like using Cassandra
  - ~100 teams writting code
  - squads: +- 7 people
  - feature squads are in charge of features - they own it end-to-end; e.g. search functionality
    - backend
    - client (iOS, Android, Web and Desktop)
    - QA
    - operational quality
    - product
  - client squads and infra squads build tools for feature squads
  - https://engineering.atspotify.com/2014/03/spotify-engineering-culture-part-1/
  - every team has the ownership, independence and responsibility over its feature
  - two code reviews per PR, in 2015 more than 800 services written in Java
    - bad thing - every squad has the same organizational problems to solve - solving it multiply times
- in early days they had ServiceDB which stored data about their services
- 2015. System-Z -> systems metadata system (they gave a dummy name just to start with which stayed there (forever)...)
- they had their own container orchestration system called Helios - https://github.com/spotify/helios
- they have a notification system in System-Z that will report any warning and red flags. They printed these reports and put them on doors of every toilet 🙂
- services have dirty metadata
