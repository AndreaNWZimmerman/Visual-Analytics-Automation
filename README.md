# Visual-Analytics-Automation
Code used to automate VA processes, relies on REST APIs

VA Automation Code Samples

1.	How do we change a single dataset within a VA report?
{
"inputReportUri": "/reports/reports/aalf-aew-q9834yr-9a8df-9a8df98",
"dataSources": [
{
   "namePattern": "serverLibraryTable",
              "purpose": "original",
    "server": "cas-test-default",
    "library": "PUBLIC",
    "table": "StartingDataSet1"
},
{
    "namePattern": "serverLibraryTable",
    "purpose": "replacement",
    "server": "cas-test-default",
    "library": "PUBLIC",
    "table": "RealDataSet1_&caseNum",
    "replacementLabel": " RealDataSet1_&caseNum "
          }
                                                ],
 
"resultReportName": "OutputReport_TEMPORARY",
"resultParentFolderUri": "/folders/folders/a8-ad-09-ad-a08fd",
"resultReport": {
"name": "OutputReport_TEMPORARY",
"description": "OutputReport_TEMPORARY"
       }
}




2.	How do we get the macro variables to resolve?

FILENAME have1 TEMP ENCODING='UTF-8';
OPTIONS PARMCARDS=have1;
PARMCARDS4;

All the code from above including the {}

;;;;

FILENAME json1 TEMP ENCODING='UTF-8';
DATA _NULL_;
   INFILE have1;
   FILE json1;
   INPUT;
   _INFILE_=RESOLVE(_INFILE_);
   PUT _INFILE_;
RUN;

The result of this is a temporary file called json1 that has the exact text of the API call we wish to make, with our macro variables resolved.





 
3.	How do we execute all that code?
%let BASE_URI=%sysfunc(getoption(SERVICESBASEURL));
FILENAME tranFile TEMP ENCODING='UTF-8';
FILENAME hdrout TEMP ENCODING='UTF-8';

PROC HTTP METHOD="POST" oauth_bearer=sas_services OUT=tranFile headerout=hdrout
URL = "&BASE_URI/reportTransforms/dataMappedReports/?useSavedReport=true&saveResult=true"
IN = json1;
HEADERS "Accept" = "application/vnd.sas.report.transform+json"
	      "Content-Type" = "application/vnd.sas.report.transform+json" ;
*debug level=2;
RUN;
 
4.	How do we get the URI for the report we just created?
FILENAME rptFile TEMP ENCODING='UTF-8';
PROC HTTP METHOD = "GET" oauth_bearer=sas_services OUT = rptFile
    URL = "&BASE_URI/reports/reports?filter=startsWith(name,"OutputReport_TEMPORARY")";
    HEADERS "Accept" = "application/vind.sas.collection+json"
		"Accept-Item" = "application/vnd.sas.summary+json";
RUN;

LIBNAME rptFile json;
/*Get the row for the most recent version*/
proc sql noprint;
  create table temp as
  select *
  from rptFile.items
  having creationTimeStamp=max(creationTimeStamp);
quit;

data ds_rpts (keep=rptID id name createdBy creationTimeStamp modifiedTimeStamp  
	             rename=(modifiedTimeStamp=LastModified creationTimeStamp=CreatedAt));
	length rptID $ 60 rptPath $ 100;
	set temp;
	rptID = '/reports/reports/'||id;
run;


%MACRO save_VA_Report_Path(reportURI);
Full code at my github link (I did not write this macro)
%MEND save_VA_Report_Path;

DATA _null_;
	set ds_rpts;
	call execute('%save_VA_Report_Path('||id||')');
RUN;

proc sql noprint;
  select rptID into :rptID trimmed
  from ds_rpts;
quit;

5.	How do we delete a report we no longer need?

PROC HTTP METHOD="DELETE" oauth_bearer=sas_services OUT=tranFile headerout=hdrout
URL = "&BASE_URI/reports/reports/&rptID";
HEADERS "Accept" = "application/vnd.sas.report.transform+json"
                  "Content-Type" = "application/vnd.sas.report.transform+json" ;
*debug level=2;
RUN;

