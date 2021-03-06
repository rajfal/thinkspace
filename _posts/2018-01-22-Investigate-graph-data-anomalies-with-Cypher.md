---
layout: post
title: "Investigate graph data anomalies with Cypher"
comments: false
description: "Cypher queries to explore data in Neo4j"
categories: [neo4j, Cypher, data analysis]
tags: [neo4j, cypher, data import]
keywords: "neo4j, Cypher, data exploration, cypher query, cypher statements, data analysis"
---

> #### *Prerequisites:*{: style="color: red"}
> - [sample soil survey data loaded into a Neo4j graph](/2018/Import-CSV-data-into-Docker-Neo4j-container/)

---

#### Background

With our soil survey data in the graph, we can start inspecting for trends of interest. As the subject is a fictional environment, we can make some rules by which this 'economy' is expected to run. We can check for various metrics and look for findings that don't meet the expected norms. We will consider such discoveries as anomalous data patterns.

For example, given a pattern like
```bash
(n:Hort_Client)<-[:SENT_TO]-(:Soil_Report)<-[:ACTIONS]-(:Contractor)-[:OPERATES_IN]->(m:Region)
```
Let's say that any horticultural enterprise must not be found to hire contracting firm from more than one Region. In fact, a contracting license specifies that a Contractor can only operate within its own Region and thus must not allow itself to be drawn into deals that would require it to work cross-border.

Another limit we can place is that a Contractor must not extend its operation across some minimum number of clients. THe idea here is that each Contractor must maintain a minimum of familiarity with a given property and can not sacrifice that aspect of its business in preference to simply maximising its income across as many clients as possible but not retaining any detailed property knowledge...

#### Exploring specific scenarios where unexpected business activities take place

> *Business rule: a `Hort_Client` can hire any `Contractor` but only from its own and one region*{: style="color: blue"}

1. Let's find hort firms that have hired contractors
  ```sql
MATCH (n:Hort_Client)<-[:SENT_TO]-(:Soil_Report)<-[:ACTIONS]-(:Contractor)-[:OPERATES_IN]->(m:Region)
WITH n.name as hort_name
MATCH path = shortestPath((n1:Hort_Client)-[*1..3]-(m1:Region))
WHERE n1.name = hort_name
RETURN DISTINCT path;
  ```
  Output:
  : - `Hort_Client` nodes with associated `Contractor` nodes. But who are the culprits ignoring the regulations?
  ![Hort_Client with many regions](/assets/images/soil_survey_hort_firm_and_contractors.png)
  
2. We can take a tabular solution to pin-pointing the related nodes of interest 
  ```sql
MATCH (n:Hort_Client)<-[:SENT_TO]-(:Soil_Report)<-[:ACTIONS]-(:Contractor)-[:OPERATES_IN]->(m:Region)
WITH n.name as hort_name, n.client as sorting, count(DISTINCT m.name) as no_regions, 
collect(DISTINCT m.name) AS region_list
WHERE no_regions > 1
RETURN hort_name, no_regions, region_list 
ORDER BY sorting;
  ```
  Output:  
  : - ```bash
╒═══════════╤════════════╤════════════════════════════════════╕
│"hort_name"│"no_regions"│"region_list"                       │
╞═══════════╪════════════╪════════════════════════════════════╡
│"hc_157"   │3           │["Northbury","Swifford","Eastling"] │
├───────────┼────────────┼────────────────────────────────────┤
│"hc_171"   │3           │["Northbury","Swifford","Westshire"]│
└───────────┴────────────┴────────────────────────────────────┘
```

3. Let's use a more incisive graph approach to show us the suspect relationships
  ```sql
MATCH (n:Hort_Client)<-[:SENT_TO]-(:Soil_Report)<-[:ACTIONS]-(:Contractor)-[:OPERATES_IN]->(m:Region)
WITH n.name as hort_name, count(DISTINCT m.name) as no_regions
WHERE no_regions > 1
WITH hort_name
MATCH path = shortestPath((n1:Hort_Client)-[*1..3]-(m1:Region))
WHERE n1.name = hort_name
RETURN DISTINCT path; 
  ```
  Output:
  : - it is *hc_157*{: style="color: red"} and *hc_171*{: style="color: red"} who've done cross-border deals
  ![Hort_Client with many regions](/assets/images/soil_survey_hort_firm_sourcing_contracts_from_many_regions.png)
  
> *Business rule: a `Contractor` can have no more than X `Hort_Client`s*{: style="color: blue"}

1. See contractors and their clients
  ```sql
MATCH (n:Hort_Client)<-[:SENT_TO]-(:Soil_Report)<-[:ACTIONS]-(c:Contractor)
WITH c.name as contractor
MATCH path = shortestPath((n1:Hort_Client)<-[*1..2]-(c1:Contractor))
WHERE c1.name = contractor
RETURN DISTINCT path;
  ```
  Output:
  : - Displaying `Contractor`s and their `Hort_Client` nodes. But are there any contractors who contravene the rules?
  ![Contractors and their Hort_Clients](/assets/images/soil_survey_contractors_and_hort_firms.png)
  

2. We can see the related nodes of interest in a table that shows us contractors  who worked for more than X clients. For this sample of records, we'll set X = 1
```sql
MATCH (n:Hort_Client)<-[:SENT_TO]-(:Soil_Report)<-[:ACTIONS]-(c:Contractor)
WITH c.name as contractor, c.c_id as sorting, count(DISTINCT n.name) as no_clients, 
collect(DISTINCT n.name) AS client_list
WHERE no_clients > 1
RETURN contractor, no_clients, clients 
ORDER BY sorting;
```
Output:  
  : - ```bash
╒═════════════╤════════════╤═══════════════════╕
│"contractor" │"no_clients"│"client_list"      │
╞═════════════╪════════════╪═══════════════════╡
│"contra_1250"│2           │["hc_160","hc_167"]│
└─────────────┴────────────┴───────────────────┘
```

3. A more direct graph view will disclose the suspect activities
  ```sql
MATCH (n:Hort_Client)<-[:SENT_TO]-(:Soil_Report)<-[:ACTIONS]-(c:Contractor)
WITH c.name as contractor, count(DISTINCT n.name) as no_clients
WHERE no_clients > 1
WITH contractor
MATCH path = shortestPath((n1:Hort_Client)<-[*1..2]-(c1:Contractor))
WHERE c1.name = contractor
RETURN DISTINCT path; 
  ```
  Output:
  : - it is *contra_1250*{: style="color: red"} who is moonshining for another `Hort_Client`
  ![Hort_Client with many regions](/assets/images/soil_survey_contra_and_two_clients.png)
  
 
 
---
***We have defined some business rules by which the data must play. We used Cypher queries to figure out where those rules are broken***{: style="color: green"}

---
[Back to top of page](#)

---



