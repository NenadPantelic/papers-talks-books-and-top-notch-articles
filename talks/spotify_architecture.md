# Spotify's Architecture

- Microservices from the beginning, 2008
- Unix principle - one simple thing, but do it great
- 13k production components (services, pipelines, frontends...) in 2023
- approx 3k lines of code per component (except for mobile)
- ~ >650 squads, roughly 6 components per squad (3500 backend services)
- ~2600 deployments per day (mobile app deployments are less frequent, once a week)
- depedencies:
  - average: depends on 9 downstream components
  - average: 9 others depend on it
- mobile decoupling - codebase growing with about ~30% year-over-year, they had monoliths
- Conway's law: design systems are constrained to produce designs which are copies of the communication structures of these organizations
- Modularity: no of external dependencies /no of internal dependencies
- Stats from 2021 - more than 500 (code) ownership changes (just for 2021)
