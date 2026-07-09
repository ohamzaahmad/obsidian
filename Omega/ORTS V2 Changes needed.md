1. [ ] ~~transition note (remove)~~
2. [ ] ~~handover notes -> notes (rename)~~
3. [ ] ~~third level request creation (parent check)~~
4. [ ] ~~Able to create the stage before the end stage~~
5. [ ] ~~instead of filtering the stage data in below in the request -> it should show the pop up for it and show all the data in the request below the request details  
    - [ ] ~~remove the stage focus (not needed)~~  
    - [ ] ~~**_need to ask how to handle the comment for individual stage_**~~
6. [ ] ~~Cancelled status~~
7. [ ] ~~due date -> core information~~ and in table after request date
8. [ ] ~~Escalated -> Due date~~
9. [ ] ~~provider in the request creation~~
10. [ ] ~~handle the tree view~~
11. [ ] ~~no stage handler in request creation~~
12. [ ] ~~column view handle, and place the due date after request date~~
13. [ ] ~~add the default values for the priority and status~~
14. [ ] ~~sidebar listing~~
15. [ ] ~~request details page responsiveness~~
16. [ ] ~~make the view and edit buttons in the stages button prominent~~
17. [ ] ~~now is another issue in the request page, requestdept page, and request details page, and request table, if the user doesn't have the last name it goes to fallback to email which it shoudln't in the request table, where the column has the user value, and in the request details~~ \> core information > ASSIGNMENT TO card 
18. [ ] ~~when i open the request details in the edit mode it shouldn't filter the assigned to filters until i first change the providers, it should first load the data from the request response and then it should show the default\_assignee of that provider if and only if i change the provider in the provider field~~
19. [ ] ~~\[maybe backend error/ resolved by backend\] there is an issue in the stage movement when i move next/previous stage, then assigned\_to in the next\_stage and previous\_stage should be automatically selected if they are in the assigned\_field of their object like this~~
20. [ ] ~~it is working for the assigned\_to only, not for the other fields which has the field type users(single) or users(multiple)~~
21. [ ] ~~NO STAGES ("stages": \[\]) AND NO ASSIGNED TO ("field\_name": "assigned\_to",) FIELD-> SHOW DEPT FIELD IN THE CREATE REQUEST FORM~~
22. [ ] ~~add the departmend\_id field in the payload and list the departments in the dropdown~~
23. [ ] ~~NO STAGES ("stages": \[\]) BUT HAVE ASSIGNED TO ("field\_name": "assigned\_to",) FIELD -> FILTER THE THE ASSIGNED TO FIELD WITH DEPARTMENT FIELD FILTER like we do using the /assignable-users/?department=~~

## PENDING CHANGES

1. [ ] don't show the show comment button if the allow\_comment is false
2. [ ] handle the provider field when there is more than one provider fields