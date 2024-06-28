POMA
POlicy Machine Analyzer: a tool for testing and verifying NGAC (Next Generation Access Control) policies.  

Verification

In order to create the obligation checker with graph and obligation(prohibition is in works), use one of the following constructors:

With objects:

Graph graph = Utils.readAnyGraph("Policies/ForBMC/LawFirmSimplified/CasePolicyUsers.json");
String yml = new String(Files.readAllBytes(Paths.get("Policies/ForBMC/LawFirmSimplified/Obligations_simple.yml")));
Obligation obligation = EVRParser.parse(yml);
ObligationChecker checker = new ObligationChecker(graph,obligation);


With paths(only single graph json supported):

ObligationChecker checker = new ObligationChecker("Policies/ForBMC/LawFirmSimplified/CasePolicyUsers.json",
"Policies/ForBMC/LawFirmSimplified/Obligations_simple.yml");

Then, you need to set the output for smt code as follows: 

checker.setSMTCodePath("VerificationFiles/SMTLIB2Input/BMCFiles/BMC1/BMC");

Additionaly, you may want to set the time horizon(default is 4) via following command:
checker.setBound(10);

Optionally, you can enable showing the SMT output(disabled by default, especially useful if you suspect an error):
checker.enableSMTOutput(true);



The solver will return null if no solution is found or Solution object that has a list of obligation firings as shown below: 

Solution [
1: ObligationFiring [obligationLabel=obligation1, subject=Vlad, event=submit, object=PDSWhole
2: ObligationFiring [obligationLabel=obligation2, subject=ChairU, event=approve, object=PDSWhole
3: ObligationFiring [obligationLabel=obligation3, subject=BMU, event=approve, object=PDSWhole
4: ObligationFiring [obligationLabel=obligation4, subject=DeanU, event=approve, object=PDSWhole
5: ObligationFiring [obligationLabel=obligation5, subject=RAU, event=submit, object=PDSWhole
6: ObligationFiring [obligationLabel=obligation6, subject=RDU, event=archive, object=PDSWhole
7: ObligationFiring [obligationLabel=obligation7, subject=UserChair, event=approve, object=PDSWhole


 Queries

 PREDICATES

In order to solve the contraint, use the following format method call: 

Solution solution = checker.solveConstraint("OBLIGATIONLABEL(obligation2);");

The solution object, hopefully, will contain a list of steps. 

The following are the queries currently supported: 
| Predicate  | Logic | Query Example | Variables Allowed? |
| ------------- | ------------- | ------------- | ------------- |
| Obligation Label is reachable  | ------ | OBLIGATIONLABEL(obligation1, Attorneys, accept, Case3Info); | YES(not for label) |
| Association exists  | (ua, ar, at) belongsTo ASSOCIATE | ASSOCIATE(Attorneys,refuse,Case3);  | YES |
| Permission exists  | (u,?ua) belongsTo ASSIGN AND (t,?at) belongsTo ASSIGN AND (?ua, ar, ?at) belongsTo ASSOCIATE | PERMIT(Attorneys2U, accept, Case3Info); | YES |
| Explicit assignment exists (no hierarchy)  | (a,d) belongsTo ASSIGN | EXPLICITASSIGN(Attorneys2U, Attorneys2); | YES |
| Implicit assignment exists (only hierarchy)  | (a,d) belongsTo (ASSIGN - ASSIGN) | IMPLICITASSIGN(Attorneys2U, Attorneys2); | YES |
| Explicit + implicit assignment exists (hierarchy accounted for) | (a,d) belongsTo ASSIGN | ASSIGN(Attorneys2U, Attorneys); | YES |
| Deny - permission does not exist | NOT(PERMIT(u,ar,t)) | DENY(Attorneys2U, accept, Case3Info); | YES |
| Hierarchy exists - either a is assigned to b or b is assigned to a(inheritance included) | (a,b) belongsTo ASSIGN OR (b,a) belongsTo ASSIGN| HIERARCHY(Attorneys2U, Attorneys2); | YES |
| Node Exists | (a,?d) belongsTo ASSIGN | NODEEXISTS(Attorneys2U); | YES |
| SUBSET | ------ | NOT NOW | ----- |

The following sets are available: ASSIGN, ASSIGN, ASSOCIATE

CONNECTIVES

Both "and" and "or" are supported. The format as follows: 

java
"(OBLIGATIONLABEL(obligation1, Attorneys, accept, Case3Info) AND ASSOCIATE(Attorneys,refuse,Case3));" 

java
"(OBLIGATIONLABEL(obligation1, Attorneys, accept, Case3Info) OR ASSOCIATE(Attorneys,refuse,Case3));"


Note that even though smt supports and/or with 3 or more elements, this feature is not included in this solver.

 NEGATION

In order to use negation, simply do: 

java
"NOT(OBLIGATIONLABEL(obligation1, Attorneys, accept, Case3Info));"


 NESTED QUERIES

The following is an example for complex queries with logical connective "AND" and negation.  

java
"((((PERMIT(Attorneys,accept,Case3Info) AND NODEEXISTS(Attorneys1)) AND NODEEXISTS(Attorneys)) 
AND PERMIT(Attorneys,?ar,?at)) AND NOT(IMPLICITASSIGN(Attorneys1,Attorneys)));"



 TERMS

Any terms that contains a "?" is considered to be a _VARIABLE_. Otherwise, it is a _CONSTANT_. Please give your variables valid names that you will recognize once the processing is done. 

 SETS

There are currently 3 sets that obligations are making changes to in SMT. 

1. ASSIGN: assignments with no hierarchy

2. ASSIGN: assignments with hierarchy

3. ASSOCIATE: associations, not permissions.

It is possible to get all the permissions with join operations, but is very expensive. Also, it is possible to get all the permissions for just ua, ar, at, or their permutations. 

If there is a need to use any of the above sets with a predicate, let me know and I will add such predicate. 

 BNF

java
"
  Parser for NGAC planner queries.
  
  Syntax: variables start with ?, predicates and constants
  with either letters or digits. 
  The binary operators "AND", "OR" must be put in parentheses; Predicates have parentheses for parameters.
  The negation operator is "NOT". Comma is used as a delimiter for predicate parameters
  The input has to end with ";"
  
  < formula > ::=  < predicate > | < binary > | <negation> {, formula} ";"
  < binary > :==  "(" < formula > "AND" < formula > ")"
           		| "(" < formula > "OR" < formula > ")"
  < negation > :== "NOT" "(" < formula > ")"
  < predicate > ::=  < PERMIT > | < ASSOCIATE > | < DENY > | < IMPLICITASSIGN > | < EXPLICITASSIGN > |
                     < HIERARCHY > | < ASSIGN > | < NODEEXISTS >
  < PERMIT >  ::=  "PERMIT" "("< term > < term > < term >")"
  < ASSOCIATE > ::=  "ASSOCIATE""(" < term > < term > < term >")"
  < DENY > ::= "DENY""(" < term > < term > < term >")"
  < EXPLICITASSIGN > "EXPLICITASSIGN""("< term > < term >")"
  < IMPLICITASSIGN > "IMPLICITASSIGN""("< term > < term >")"
  < ASSIGN > ::=  "ASSIGN""("< term > < term >")"
  < HIERARCHY > ::= "HIERARCHY""(" < term > < term >")"
  < NODEEXISTS > ::=  "NODEEXISTS""("< term >")"
  < term >  ::= CONST | VAR
"





 Current Limitations
Only the following obligation actions are currently supported: Create Node, Add Assignment, Remove Assignment, Add Association, Remove Association.

Some actions have limitations as follows: 

Create Node: only 1 node can be created per single action.

Add Assignment: only 1 assignment can be created per single action.

Remove Assignment: only 1 assignment can be removed per single action.

Add Association: only 1 association can be added with only 1 access right per single action.

Remove Association: only 1 association can be removed with only 1 access right per single action.

Each of these limitation can be solved by simply creating multiple actions. For example, the association (UA, {ar1, ar2}, AT) can be instead written as two: (UA, ar1, AT) and (UA, ar2, AT).



 User Interface

We provide a friendly user interface that can be used for verification and testing of NGAC policies.
When the user starts POMA, they will be presented with the following screen. 
Alt text(docs/imgs/1.png)

 Visualization

The left hand side lets the user pick the policy. Usually, policies are stored under Policies folder within the project folder. Each policy should have their own folder. For visualization purposes, the policy should have at least graph. Subfolder are allowed as well(please keep in mind the all the policies within folders and subfolders will be loaded on selecting a folder). 

Alt text(docs/imgs/2.png)

Once the policy within the folder is loaded(as in the above image), the graph will be visualized and will include all the policy classes files within the folder. Note that you cannot edit the policy if it was loaded from folder. You can, however, edit and save the policy if you pick a single json file as follows: 

Alt text(docs/imgs/3.png)

 Verification

POMA provides several approaches for verification of policies. To start, select a policy from the file tree you intend to verify(e.g. 'GUI Test/simple').

 Reachability Analysis

Once you select a policy, in the top menu choose Verify -> Reachability Analysis. The following tab will appear: 


Alt text(docs/imgs/4.png)

Reachability analysis is used to verify whether the goal property can be reached within the number of steps. The goal property can be defined using the same predicates described in the [Queries(./README.mdqueries) section above

For the testing of GUI Test/simple, let's enter the following in the post goal:  


OBLIGATIONLABEL(obligation2,?u,?ar,?at);


We also want to select the time horizon as 2 (as we know prior that we will not be able to reach the obligation within horizon of 1).

After we click Run (and a few seconds later), we will get the following solution:

Alt text(docs/imgs/5.png)

The solution includes each obligation that is required to be executed to reach the goal property as well as any custom variables(either from query or from custom obligation functions). We also can optionally specify the property that has to be help prior to the final obligation execution in the left text box(pre property). 

As specification of queries can be an overhead, we provide the ability to specify them with 2 property files(and include them within the policy folder). The names of such files must be pre.property and post.property for pre and post property respectively. If these files exist, the property inputs will be prepopulated as can be seen when opening reachability interface of GUI Test/PrePostProvided:

Alt text(docs/imgs/6.png)

Alt text(docs/imgs/7.png)

We also provide the ability to provide the customization.txt file with custom functions specification.
Custom functions are usually used to query elements to further use within actions(more in paper).

The following is an example of add_copi obligation with custom functions(note you need to specify action_index, where the index of the starts with 0):


CustomFunctions: add_copi
	query: (((((((((EXPLICITASSIGN(?copi_to_add, ?department) AND EXPLICITASSIGN(?department, ?chairattribute)) AND EXPLICITASSIGN(?chair, ?chairattribute)) AND EXPLICITASSIGN(?chairattribute, ?bmattribute)) AND EXPLICITASSIGN(?bm, ?bmattribute)) AND ISUSER(?chair)) AND ISUSER(?bm)) AND EXPLICITASSIGN(?bmattribute, ?deanattribute)) AND EXPLICITASSIGN(?dean, ?deanattribute)) AND ISUSER(?dean)); 
	action_0:
		action: assign(?chair, Chair);
	action_2: 
		action: assign(?bm, BM);
	action_3: 
		condition: NOT(EXPLICITASSIGN(?dean, Dean));	
		action: assign(?dean, Dean);
	action_4: 
		condition: NOT(EXPLICITASSIGN(?dean, Dean));		
		action: grant(PI, submit, ?test);	



The following files are required for the reachability analysis:

- .json: Graph represented in JSON format
- .yml: Obligations represented in YML format

The following files can be optionally provided for the reachability analysis:

- pre.property: file with pre.property specification
- post.property: file with post.property specification
- customization.txt: custom function specification for obligation


 Initial configuration analysis

Similar to reachability analysis, we can perform the analysis of a static configuration without obligations by selecting Verify -> Initial configuration analysis

There will be just one input window for property (which can be prepopulated with pre.property file) and the solution will either be satisfied or unsatisfied as follows: 

Alt text(docs/imgs/8.png)

The following file is required for the initial configuration analysis:

- .json: Graph represented in JSON format

The following file can be optionally provided for the initial configuration analysis:

- pre.property: file with pre.property specification


 Race condition analysis

Race condition analysis finds obligations that can be conflicting if in different order. In order to find race condition, we look at each order for the obligation with similar events and output the combination leading to the race condition. To start, select the GUI Test/simple and follow to Verify -> Race condition analysis.

Select time horizon of 2 and max combinations of 2(how many permutations of obligations we want to check, we need to bound as total number is n! where n is number of obligations). Click Run.

Alt text(docs/imgs/9.png)

We will find 2 solutions and the combinations with variables which led to race condition.

The following file is required for the initial configuration analysis:

- .json: Graph represented in JSON format
- .yml: Obligations represented in YML format

The following file can be optionally provided for the initial configuration analysis:

- customization.txt: custom function specification for obligation

 Initial policy permissions

Finally, the Verify -> initial policy permissions option can be used to quickly analyze and output all the access permissions within the policy as follows: 

Alt text(docs/imgs/10.png)

