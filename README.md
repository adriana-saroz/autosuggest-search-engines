Autosuggest Search Engines
==========================


Project Proposal
----------------
The Firefox search bar provides the ability to add a website as a search engine, while visiting a website that complies to the OpenSearch standard. Although most websites offer search functionality, only few present themselves as OpenSearch capable.
This project aims at adding to Firefox the ability to detect repeatedly used search fields and suggest websites as being search capable. The result from an user experience point of view will be seeing an entry with "Add <website>" under the search engines list in the search bar. By choosing to add the suggested entry, users will be able to query a detected search field without having to first visit the source website.

The detection will take place when the user submits a query through a form. The form will be validated against a set of criteria in order to decide if that form has the characteristics of a search form or not. I have split the design/implementation of the algorithm into two phases. 

Phase 1 is a simple set of yes/no criteria that decide whether a form qualifies or not. The set of criteria is the following:
The following should occur for the form to be considered a candidate:

* The form contains one visible input of type search or text.
* Note that the form may contain one or more inputs of type hidden.
* The form method is get or post.
* The form may contain inputs of type combo, only if these have a default value.

Any of the following exclude the form from being a candidate:

* The site is already added to the list of managed search engines.
* The form contains more than one visible input of type search or text. (probably registration form or survey)
* The form contains at least one input of type password. (probably login form)
* The form contains at least one input of type email. (probably registration form)
* The form contains at least one input of type file. (probably file upload. There exist image based search engines, but this is a special case that I would not handle as part of GSoC)
* The form contains a visible textarea (probably blogs, status, comments)

Hypothesis testing of the algorithm  

* Facebook  
    * Use Cases that qualify:  
        * The search form in the header qualifies.  
    * Use Cases that do not qualify:
         * The login form on the front page does not qualify because the password input is of type password.
         * The sign up form on the front page does not qualify for several reasons: it contains inputs of type password, it contains more than one input of type text.
         * The Update Status form contains an input of type text for tagging people (who-are-you-with) but it also contains a textarea
         * The comments sections contain textareas.  

* Reddit  
     * Use Cases that qualify:
         * The search form has one single text input and does not contain unallowed elements.  
     * Use Cases that do not qualify:  
         * The login form right below the search form.

The problem with phase 1, is that any form that contains only one input of type text and no other elements, would qualify. For example, the pbs portal (www.pbs.org) has a search form in the header, but it also has a subscription form (with input type text). Both forms would qualify against the above criteria. Now, in this particular case a user would probably not use the subscription form more than one time, but this example proves the flaw of the above algorithm.

Which is why, for phase two I would enhance the above algorithm by adding a ranking mechanism. All of the above arguments remain valid and they will be assigned with corresponding positive/negative weights. Additionally, I would add weights to the following factors:

* Positive weight if the input with type text has onkeyup/onkeydown event handlers. They might indicate the presence of incremental search or real-time suggestions.
* Positive weight if the input, the form or any of the intermediary html elements have ids/classes/titles that contain the word "search". This can also be extended by adding common international equivalences of the word "search". Note that not meeting this criteria would not reduce the rank of a candidate form.
* Positive weight if the absolute position of the form is smaller than some threshold, thus indicating that the form might be part of the header of the website (most search fields are part of headers).


FormSubmitListener.js catches the action of submitting a form, therefore the code for the detection algorithm will run inside of the notify function. If the form is detected to be a search form, an async message will be sent in order to notify the form history prototype component that a search form has been detected:
    sendAsyncMessage("FormHistory:SearchFormDetected", data);

When initializing FormHistory.prototype inside nsFormHistory.js, a listener will be added for the above message:
    this.messageManager.addMessageListener("FormHistory:SearchFormDetected", this);

The form history sqlite database that is saved locally inside of the firefox profile folder is currently used for form auto-completion. It's code can be found in the nsFormHistory.js file. This database can be tweaked to also store the necessary information for the search engine auto-suggest feature.

First, I would add a column for the website url to the moz_formhistory table. For implementing the detection algorithm I would apply the following: If x >= y and the detection algorithm validates the form, than the form is considered to be a search form.  
x = number of times an input was used = SELECT SUM (timesUsed) FROM moz_formhistory where fieldname = <name> and url = <url>  
y = number of times an input should be used in order to be considered

Secondly, I would add a new table to the database, moz_searchforms, that should contain an entry for every form/website that was detected. This table would be queried on page load in order to append "Add <website>" in the search bar for a form that was detected during a previous session. This table will contain columns for hostname, title, search url, search params, and any other data that is otherwise provided by the OpenSearch description xml (which is used when creating a search plugin for Firefox).

The changes that will be brought to the nsFormHistory.js file include:

* dbSchema update: url column for moz_formhistory
* dbSchema update: moz_searchforms table
* Adding a dbMigrateToVersion(5) function
* Necessary operations for moz_searchforms: function for adding an entry and removing entries (removal will be synchronized to the removal from the moz_formhistory table)
* Updating addEntry function to have another parameter for website url (also update insert and update sql statements)

The third major change is adding a mechanism to use the data from the moz_searchforms tables instead of the data provided by open search xmls in order to generate the search plugin xmls and update search.json (the profile file that holds the list of installed search plugins)


Schedule of Deliverables
------------------------

Between the time Google announces the list of accepted students and until the final evaluation submission date, there is a work period of 17 weeks. I will be attending classes and have my thesis due by the 3rd of July, so I would be able to work about 2-3 hours/day until then, and full time after that as I have no other commitment. Although this means only 11 weeks of full time work (and 6 of part time which I do not consider lost because I will be using this project as a context switch from the school work that I will be doing), I am confident that I can successfully complete the project.

I estimate my project schedule to be the following:

* May 28 - June 9 (2 weeks) Get to know the community, study the code base, discuss the project details and any design issues that might arise with the mentor.
* June 10 - June 23 (2 weeks) Update dbSchema, implement add/update and query functionality for the new db structure.
* June 24 - July 14 (3 weeks) Implement the simple algorithm described for phase 1 and test it against a larger number of websites. Write some simple tests to validate the algorithm. Note: Up until July 3rd I will be working in paralel with my thesis.
* July 15 - July 21 (1 week) Hook up the form detection with the UI: append an entry with "Add <website>" under the search engines list in the search bar and any additional minor details that need to be done by midterm.
* Mid-term deliverable: A working version of the simplified search field detection algorithm.
* July 22 - Aug 4 (2 weeks) Implement and hook up mechanism to use moz_searchforms data instead of open search xml.
* Aug 5 - Aug 18 (2 weeks) Implement the extended algorithm described for phase 2.
* Aug 19 - Aug 25 (1 week) Implement database remove entry / clear cache functionality.
* Aug 26 - Sept 8 (2 weeks) Write tests for the extended detection.
* Sept 9 - Sept 22 (2 weeks) Code cleanup and improve documentation.

A stretch goal in case everything works ahead of schedule is to discuss any necessary UX improvements with the mentor and proceed with implementing the discussed changes during or after GSoC.


Open Source Development Experience
----------------------------------
While working on a web application based on the SproutCore framework (http://sproutcore.com/), I have reported several bugs. I have also brought improvements to the version that we had forked at the time (about 2 years ago). The most significant ones were increasing the accuracy of gestures recognition when dealing with swipe and scroll simultaneously and improving the algorithm used for recognizing the user agent.


Work/Internship Experience
--------------------------
I have recently left my job as a software developer for 3 Pillar Global (https://www.3pillarglobal.com/) in order to finish my master's degree and devote some time to other projects like GSoC. During the two and a half years that I’ve spent there I worked in the context of web and native mobile development. I was part of three major projects and teams, and I also switched between different technologies: JavaScript, Python and Android. I have a few months worth of JavaScript work experience as part of the UI team for a news publishing web application, based on SproutCore and I have done both frontend and backend work for the mobile website of the PBS video portal and other PBS products. I also worked as an intern for TBA (http://www.tba.nl/), where I developed a 3D driver application that works synchronized to Google Earth and allows viewing replays of different large scale simulations.


Academic Experience
-------------------
I graduated with a Bachelor degree in Computer Science at the Babes Bolyai University in Cluj Napoca, Romania (http://www.cs.ubbcluj.ro/en/), in 2011, where I was ranked 9/150. I am currently pursuing a masters program in Distributed Systems at the same university. I am working on my last courses and my thesis and I'm expected to graduate in July.


Why Me
------
I already have hands-on experience with the technologies needed in order to complete this project. I have worked with JavaScript before, also with databases in general and SQLite in particular while developing on Android. I am familiar with joining a large team and a project with a very large code base as I have done this two times in the past and I am very excited by the prospect of digging through a huge JavaScript codebase and adding features to it.
