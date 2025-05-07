# Scaling the Data Infrastructure @Spotify

- Big Data -> Hadoop
- started with 5 servers in the office 2007; an interesting problem - servers were overheating because the sunshines were hitting them. They bought some curtains.
- tried Amazon Elastic Map/Reduce in 2010
- 200 nodes in 2012
- they started running a 2000+ node cluster in London in 2017; running out of the physical space
- 12000 servers
- moved to Google Cloud Platform
- 2017 stats:
  - over 100 milion monthly active users
  - over 20 milion songs
  - over 2 billion playlists
  - active in 60 markets
  - about 50 TBs/day
  - data analysts produce 60 TBs/day; 10k Map/Reduce jobs
- Summer of 2015 = a strain of incidents
  - too many Hadoop jobs, they DDoS-ed themselves; edge nodes were scheduling those jobs
  - Hadoop keeps data and processing at the same node
- Challenges and the path to victory

  1. early warning -> datamon - data monitoring
  2. debuggability & control -> Styx - scheduling & control; SystemZ - a way to control (turn on/off a component)
  3. automate capacity -> GABO - event delivery

- Data management problems they had
  1. unified view
  2. ownership
  3. SLA
