chiliproject-to-redmine
=======================

Chiliproject to Redmine migration howto and resources


To migrate from Chiliproject 3.x back to Redmine 2.1 we used the guide provided by http://code.google.com/p/chili-to-redmine/

But we faced with several issues which can be solved with the howto and resources provided here.


Issues
------

After the database migration the following issues occurred:

*  old / obsolete paths: /chiliproject/
*  issues cause internal server errors (template parsing errors)
*  plugins broke (ChiliProject uses Rails 2.x, but Redmine 2.1 Rails 3.x)
*  themes broke (CSS ids changed and CP menu structure is incompatible to Redmine)


Post-processing
---------------


