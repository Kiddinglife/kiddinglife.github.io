---
layout: post
title:  "Good Logging Practices"
date:   2016-03-22 21:12:03
categories: logging
tags:  debug 
---

* content
{:toc}

#### 1 Good or Bad Logging
How long does it take to find the root cause of a failure in your system?
 - tens of minutes  
   - it’s very likely that your production system and tests have great logging
 - tens of hours  
   - it’s very likely that your production system and tests have poor logging

#### 2 Never log too much 
If you log too much, you’ll need to:  
- devise complex approaches to minimize disk access,  
- maintain log history, archive large quantities of data, 
- query these large sets of data. 
- make it very difficult to find valuable information.  
      
#### 3 Never log too little
normally two main goals of logging:  
- help with bug investigation 
- and event confirmation.  

If your log can’t explain the cause of a bug or whether a certain transaction took place,  
you are logging too little.
   
#### 4 Good things to log
- Important startup configuration
- Errors
- Warnings
- Changes to persistent data
- Requests and responses between major system components
- Significant state changes
- User interactions
- Calls with a known risk of failure
- Waits on conditions that could take measurable time to satisfy
- Periodic progress during long-running tasks
- Significant branch points of logic and conditions that led to the branch
- Summaries of processing steps or events from high level functions.  
  Avoid logging every step of a complex process in low-level functions.    
      
#### 5 Bad things to log    
- Function entry.  
   - Don’t log a function entry unless it is significant or logged at the debug level.
- Data within a loop.  
   - Avoid logging from many iterations of a loop.    
   - It is OK to log from iterations of small loops or to log periodically from large loops.
- Content of large messages or files.  
   - Truncate or summarize the data in some way that will be useful to debugging.
- Benign errors.  
   - Errors that are not really errors can confuse the log reader.    
   - This sometimes happens when exception handling is part of successful execution flow.
- Repetitive errors.  
   - Do not repetitively log the same or similar error.  
   - This can quickly fill a log and hide the actual cause.  
   - Frequency of error types is best handled by monitoring.  
   - Logs only need to capture detail for some of those errors.
- Useless infos.
            
#### 6 More Than One Logging Level
- You can enable certain levels at system startup, which provides a convenient control for log verbosity.
- Practically speaking, only two log configurations are needed:
  - Production  
    - Every level is enabled except debug.  
    - If something goes wrong in production, the logs should reveal the cause.
  - Development & Debug  
    - While developing new code or trying to reproduce a production issue, enable all levels.
- The classic levels are:
    - Debug  
      - verbose and only useful while developing and/or debugging.
    - Info  
      - the most popular level.
      - should be helpful to locate warn,error and critical msg.
      - usually should be <= 10% of debug or trace logs.
      - eg. subsystem initializations or a request succeeds.
    - Warning  
      - strange or unexpected states that are acceptable.
      - but if not solved, it may become normal or critical error
      - eg. disk usage exceeding throuds.
    - Error
      - something went wrong, but the process can recover.
    - Critical  
      - the process cannot recover, and it will shutdown or restart.  
      - Usually, the process logs only one critical msg once shutdown or restart.

critical and error msgs are generated when application itself goes wrong  
and problems need to be fixed by external forces like developer or system admin.

the process actually died if critical msg happens while the process is still alive  
but cannot provide service if error msg happens.
 
#### 7 More Than One Logging Category
The classic categories are (based on functionality):
- Diagnostic log
  - network request entry and exit
  - external service invocation and return
  - resource operation: open file handler
  - error-tolerant operation
  - program error: fail to connect to db
  - background operation: resource clean up 
  - startup and close process and load configures
  - do not log when exception thrown
- Statistical log
  - user visits
  - billing log
  - resource log: network or disk usage
- Monitoring log
  - administration operation
 
Statistical log and Diagnostic log complement each other  
 - Statistical log shows the symptoms of problems and Diagnostic  
 - log shows details and state on individual transactions.
   
For each log type above (request entry and exit and so on),  
they should:  
 - write into different log files
 - log different msgs with different formats

#### 8 Use Unique Identifiers  
create unique identifiers for transactions that involve processing across  
many threads and/or processes to  makes it much easier to trace a specific  
transaction when many transactions are being processed concurrently.
 1. initiator of the transaction should create the ID
 2. initiator passes id to every component that performs work for the transaction
 3. the id is logged by each component when logging information about the transaction

#### 9 Log Timing Infos
Logged timing data can help debug performance issues  
 - eg. determine the cause of a timeout in a large system  
 
Logging the start and finish times of calls that can take measurable time:
 - Significant system calls
 - Network requests
 - CPU intensive operations
 - Connected device interactions
 - Transactions

#### 10 Use Log Queue
 - When errors occur, the log should contain a lot of detail. 
 - Unfortunately, detail that led to an error is often unavailable once the error is encountered
 - A good way to solve this problem is to create temporary, in-memory log queues.
   1. Throughout processing of a transaction, append verbose details about each step to the queue.
   2. If the transaction completes successfully, discard the queue and log a summary.
   3. If an error is encountered, log the content of the entire queue and the error.
 - This technique is especially useful for test logging of system interactions.
 - An implementation for log queue please see [https://github.com/egymgmbh/log-queue] 
 
#### 11 Always Improve Logging
When problems occur, before fixing the problem:  
 - fix your logging so that the logs clearly show the cause. 
    - in development env
      - while writing new code, try to refrain from using   
      debuggers and only use the logs. 
      - Do the logs describe what is going on? If not,the logging is insufficient.
    - in production env 
      - if you cannot reproduce the problem, or you have a flaky test,   
      enhance the logs so that the problem can be tracked down when it happens again.
 - If this problem ever happens again, it’ll be much easier to identify.







    
     
    
      
    
    
       
       
    
    
 



