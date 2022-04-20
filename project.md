# Dynamic query builder for Facebook analytics

## Dashboard
* Different charts, line, pie, single value
* Different granularities, daily, weekly, monthly, summation
* future scale, 100 metrics, 10 different tables
* Analytics can be done per account or ad group

## Architecture
https://querybuilder.js.org/

* Queries can be grouped by OPERATOR, group of queries called Group
* Each group can have a subgroup
* Each group will have entries in sequence
* Group can perform operations on returned Events and subgroups within it
* Each entry: fetches row according to filter
* Operation in each entry will be field dependent


----
## Entry
class Entry:
	* table:table_name excluding granularity
	* field: field_name
	* op: <in>/<not in>/<equals>/<greater>/</less>/<greater than equal to>/<less than equal to>
	* value: value
	* group_by: account/ad
	* granularity: daily/weekly/monthly

	* fetch: function to return rows, create cache entry
	* Notes:
		* Returns a list of rows
		* Each fetch creates a cached entry, with a reference made by <table>.<granularity>.<field>.<op>.<value>.<group_by>

## Group
class Group:
	* name: a unique name for the group if any, else randomly generated
	* op: AND/OR/None/ADD/SUB/DIV/MUL
	* group_by: account/ad, this will get passed to entry
	* granularity: daily/weekly/monthly, get passed to entry
	* entries: list of Entry
	* groups: list of subgroups
	* fetch: function to return rows, create cache entry
	* eval_field: single field to use in calculation
	* eval_function: function SUM/SUB/AVG etc.

	* notes:
		* The 'op' is applied to output of each entry and of each subgroup
		* Each fetch creates a cached entry, with a reference made by to each subgroup, events, op, granulariry, eval_func, eval_field

## Query
class Query:
	* name: query name
	* type: event/query
	* event_eval 
	* group: the initial group
	* group_by: account/ad, this will get passed to group
	* granularity: daily/weekly/monthly, get passed to group

## Notes:
	* Each Event is independent, so can eaily do async/wait
	* Each Event creates a unique reference, that can be checked in cache and waited on
	* Indexes for 	
		* event_group
		* event_subgroup
---
## Examples

### Facebook Ad Clicks
 FacebookAdData_ * Sum(clicks)
 	Query:
		name: Facebood Ad Clicks
		group:
			Group:
				name: G1
				op: none (ignored as no other group)
				groups: []
				eval_field: clicks
				eval_function: SUM
				entries: [
					Entry:
						table_name: FacebookAdData
				]

### Facebook Ad Impressions
	FacebookAdData_* Sum(impressions)
 	Query:
		name: Facebood Ad Impressions
		group:
			Group:
				name: G1
				op: none (ignored as no other group)
				groups: []
				eval_field: impressions
				eval_function: SUM
				entries: [
					Entry:
						table_name: FacebookAdData
				]

### Facebook Ad Cost Per Click 
	FacebookAdData_* Sum(spends) / Sum(clicks)
 	Query:
		name: Facebook Ad Cost Per Click
		group:
			Group:
				name: G1
				op: DIV
				groups: [
					name: G1.1
					op: none
					eval_field: spends
					eval_function: SUM
					entries: [
						Entry:
							table_name: FacebookAdData
					],

					name: G1.2
					op: none
					eval_field: clicks
					eval_function: SUM
					entries: [
						Entry:
							table_name: FacebookAdData
					],

				]

### Facebook Ad Orders Per Click
	FacebookAdOrders_* Sum(quantity) / FacebookAdData_* Sum(clicks)
 	Query:
		name: Facebook Ad Orders / click
		group:
			name: G1
			op: DIV
			groups: [
				name: G1.1
				op: none
				eval_field: quantity
				eval_function: SUM
				entries: [
					Entry:
						table_name: FacebookAdOrders
				],

				name: G1.2
				op: none
				eval_field: clicks
				eval_function: SUM
				entries: [
					Entry:
						table_name: FacebookAdData
				],

			]

### Facebook Ad Event 

	(A + B) / C, where:  
	A = FacebookAdEvents.where(event_group="A" and (event_subgroup="A" OR event_subgroup="C"))  
	B = FacebookAdEvents.where(event_group="B" and event_subgroup="C")  
	C = FacebookAdEvents.where(event_group="C" and (event_subgroup="A" OR event_subgroup="C"))  

 	Query:  
		name: Facebook Ad event
		group:
			Group:
				name: G1 (A+B)/C
				op: DIV
				groups: [
					Group:
						name: G1.1 A+B
						op: PLUS
						groups: [
							Group:
								name: A event_group="A" and (event_subgroup="A" OR event_subgroup="C")
								op: AND
								groups: [
									Group:
										name: A.1
										op: none
										entries: [
											Entry:
												table_name: FacebookAdEvents
												field: event_group
												value: "A"
										],
									
									Group:
										name: A1.2 (event_subgroup="A" OR event_subgroup="C")
										op: OR
										entries: [
											Entry:
												table_name: FacebookAdEvents
												field: event_subgroup
												value: "A"

											Entry:
												table_name: FacebookAdEvents
												field: event_subgroup
												value: "C"
											]
								],

							Group:
								name: B event_group="B" and event_subgroup="C"
								op: AND
								entries: [
									Entry:
										table_name: FacebookAdEvents
										field: event_group
										value: "B"

									Entry:
										table_name: FacebookAdEvents
										field: event_subgroup
										value: "C"
								],
									
					Group:
						name: C event_group="C" and (event_subgroup="A" OR event_subgroup="C")
						op: AND
						groups: [
							Group:
								name: A event_group="C"
								op: none
								entries: [
									Entry:
										table_name: FacebookAdEvents
										field: event_group
										value: "C"
								],
									
							Group:
								name: C1 (event_subgroup="A" OR event_subgroup="C")
								op: OR
								entries: [
									Entry:
										table_name: FacebookAdEvents
										field: event_subgroup
										value: "A"

									Entry:
										table_name: FacebookAdEvents
										field: event_subgroup
										value: "C"
									]
								],
					]

				]

