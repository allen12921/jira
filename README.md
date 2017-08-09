# jira 7
CREATE DATABASE jira CHARACTER SET utf8 COLLATE utf8_bin;  
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,DROP,ALTER,INDEX on JIRA.* TO '<USERNAME>'@'<JIRA_SERVER_HOSTNAME>' IDENTIFIED BY '<PASSWORD>';  
flush privileges;


# import project only
you will need clear up the label table and also check if the xml file contains any space in <Label value
also copy the attachments to new jira_home/import/attachments/

# manually add sprint infor for ticket after imported ticket
copy over AO_xxxx_SPRINT and AO_xxxx_AUDITENTRY tables to new jira db
Grab a mapping of the RAPID_VIEW_ID's for all your boards in OD vs. your Production Instance (you can grab this by navigating to the board & grabbing it from the URL)
Change the entries in the AO_xxxxx_SPRINT table so that the RAPID_VIEW_ID's align with your Production Instance, something like
UPDATE "AO_60DB71_SPRINT" set "RAPID_VIEW_ID" = 6 WHERE "RAPID_VIEW_ID" = 3;
UPDATE "AO_60DB71_SPRINT" set "RAPID_VIEW_ID" = 5 WHERE "RAPID_VIEW_ID" = 12;
UPDATE "AO_60DB71_SPRINT" set "RAPID_VIEW_ID" = 4 WHERE "RAPID_VIEW_ID" = 13;

Back on the Standalone system we now need to grab all the customfieldvalues for all the Sprint fields for the old issues, we're going to use this to edit all the post imported issues later to populate out boards
You need to know 2 things: the custom field id of the source system & the destination for the Sprint field
In my systems this was 10007 for the source & 10306 for the destination (you can get this from the Custom fields page in the Admin area of JIRA by clicking on edit or view, grabbing it from the URL) 

for l in  "`mysql -ujira -pjira jira -e "select CONCAT(p.pkey,'-', i.issuenum),c.stringvalue from customfieldvalue c join jiraissue i on c.issue = i.id join project p on i.project = p.id where c.customfield = 10021" | sort -k 1`"
do
echo $l >>sprint.txt
done
awk '{a[$1]=$2","a[$1]}END{for (i in a){ gsub(/,$/,"",a[i]);print i,a[i]}}' sprint.txt >sprint.txt.multiplesprints
while read id sprint ; do echo curl -D- -u \$\{u\}:\$\{p\} -X PUT -H \'Content-Type: application/json\' -d \'{ \"fields\" : { \"customfield_10021\": \"${sprint}\" } }\' \$\{h\}/rest/api/2/issue/${id}; done <sprint.txt.multiplesprints
