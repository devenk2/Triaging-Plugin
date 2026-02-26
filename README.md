
A common problem in application security today, specifically with static code analysis, is that scans can result in thousands of “findings”, which need to be triaged as false or true positive. The issue that AppSec teams often face is that although these scans are necessary to find vulnerabilities in code and meet security requirements, they do not have the bandwidth or resources to efficiently triage these findings as they arise. Furthermore, pressure to release new features quickly often forces teams to skip the analysis of these scan results. We are looking to create a plugin for Claude which can effectively triage SAST results, allowing developers to code efficiently while Claude focuses on security.
 
How a security analyst would do this job today:
Export scan results from the SAST tool and open the code repository, or review within the tool if possible
Prioritize scan results, focusing on Critical and High ratings first
For each finding:
Use knowledge of code security to determine if the finding is a true positive and needs to be remediated, or a false positive and should be dismissed
If unsure, reach out to the development team for more context
Mark the findings as true or false positive
Compile the results, and communicate them to the development team for remediation
 
Inputs to our plugin:
SAST scanner findings (particularly from AppScan)
Code repository which was scanned
Context about the repository, including but not limited to:
What it does
Languages, frameworks, and tech stack
Hosting frameworks
Internal or externally facing
 
Outputs:
For each finding in the report:
True/False positive
Brief explanation why
Brief remediation advice
A full report with all triaged results