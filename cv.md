---
author: Konrad Siek
title: CV
date: konrad.siek🐌gmail.com
output: 
    html_document:
        css: css/davidwhipp-screen.css
        self_contained: no
layout: cv
---

<!-- compile:
     pandoc -s cv.md -o cv.html -c css/davewhipp-screen.css 
-->

<link rel="stylesheet" href="css/davidwhipp-screen.css">

<br/>

## Summary

I write code, I write prose, I do sketchy things to runtimes  
I solve simple problems in complicated ways  
I like working with students  

## Education

`January 2017` 
**Doctorate** in **Computing Science** at [Poznań University of Technology](http://www.put.poznan.pl/)  
Dissertation: *Distributed pessimistic transactional memory: algorithms and properties* [`🔗`](https://kondziu.github.io/pub/dissertation.pdf)  
Advisor: Paweł T. Wojciechowski  
Reviewers: Marek Tudruj, Michel Raynal

`September 2009`
**Master’s** in **Computing Science** (Software Engineering) at [Poznań University of Technology](http://www.put.poznan.pl/)  
Thesis: *A Java source code precompilation tool for static analysis and modification of programs for the Atomic RMI library*  
Advisor: Paweł T. Wojciechowski  

`February 2008`
**Bachelor of Engineering** in **Computing Science** at [Poznań University of Technology](http://www.put.poznan.pl/)  
Thesis: *Amebae: a group instant messenger for developers* (co-author)  
Advisor: Bartosz Walter

`June 2007`
**Bachelor of Arts** in **English Philology** at [PWSZ in Piła](https://puss.pila.pl/)  
Thesis: *Computer-assisted language learning software: experimental study*   
Advisor: Anna Szczepaniak-Kozak  

## Employment

`2017–2022`
**Post-doc researcher** at [Programming Research Lab](https://prl-prg.github.io/) at [Czech Technical University in Prague](https://www.cvut.cz/)  
‣ Analysis of large code repositories  
‣ Larger than memory object abstraction for R  
‣ R runtime internals survey  
‣ Teaching

`most of 2017`
**Visiting researcher** at [Programming Research Lab](http://prl.ccs.neu.edu/) at [Northeastern University](http://www.northeastern.edu/)  
‣ Lazy evaluation in R  
‣ Teaching 

`2013–2017`
**Research assistant** at [Distributed Systems Group](http://dsg.cs.put.poznan.pl/) at [Poznań University of Technology](https://www.put.poznan.pl/)  
‣ Transactional memory safety properties  
‣ Distributed TM system implementation and benchmarking  
‣ Static analysis and code generation  
‣ Teaching  

`2009–2012`
**Developer** for IT-SOA Research Project at [Poznań University of Technology](https://www.put.poznan.pl/)  
‣ Static analysis of critical sections  
‣ Code generation and Java bytecode instrumentation  

`2008–2009`
**Developer** for [PSI Poland](https://www.psi.pl)  
‣ Database stuff for an automotive factory

`2005–2006`
**Apprentice English Language Teacher** at [Elementary School No. 4 in Piła](https://sp4.e-pila.pl/)  
‣ Teaching (under supervision)

<!--`2007`
**Volunteer** for (District Municipal Library in Piła)(https://www.biblioteka.pila.pl/)-->

## Projects

`February 2019`
**UFOs:** Lazy larger-than-memory object arrays via userfaultfd  
[`🔗`](https://github.com/ufo-org/)
User provides an arbitrary function to populate a chunk of memory. 
Framework allocates an area of memory and transparently executes the population function when a chunk is read or written to.
Chunks are seemlessly garbage-collected and re-generated as needed. 
Example implementations generate in-memory arrays from columns in CSV files, BZIP file, and formulas.
Comes with C, R, and Rust bindings.  
My contribution: Back-ends, R bindings and utilities, parts of garbage collection. 

`July 2020`
**CodeDJ:** Reproducible queries over large-scale software repositories  
[`🔗`](https://codedj-prg.github.io)
Infrastructure for querying GitHub and similar repositories for quantitative software engineering research (especially project selection) in large code datasets. 
It prioritizes reproducibility and scalability and consists of two modules. 
*Parasite* is an incremental downloader and persistent datastore. 
*Djanco* is an in-memory database and query language embedded in Rust.  
My contribution: Djanco.

`February 2021`
**FML:** A small runtime for teaching runtimes  
[`🔗`](https://github.com/kondziu/FML)
Toy bytecode compiler and interpreter designed as a model for student implementations in a runtimes class.
Runs a vaguelky ML-like toy dynamic language with objects, inheritence, dynamic dispatch and garbage collection but not much else. 
The compiler generates slightly extended Feeny bytecode (another teaching language) consisting of 17 ops and 7 internal objects.  
My contribution: Solo project.

`July 2021`
**Rust-delegate:** Method delegation generator macro for Rust  
[`🔗`](https://github.com/Kobzol/rust-delegate)
A Rust macro that generates method delegation to inner fields within structs.  
My constribution: Syntax for injecting arbitrary expressions as arguments ot delegated functions.


`July 2018`
**TinyTracer:** A minimalistic tracer for analyzing the composition of R objects  
[`🔗`](https://github.com/PRL-PRG/tinytracer/)
R 3.5 runtime variant instrumented to analyze objects at garbage collection. 
The tracer records the types of each object, and the types object in all the slots slots in each object. 
Used to find rare and anomalous object constuction.  
My contribution: Everything.

`2017–2018`
**R-dyntrace:** 
[`🔗`](https://github.com/PRL-PRG/R-dyntrace)

`January 2019`
**GHGrabber:** Small GitHub scraper
[`🔗`](https://github.com/PRL-PRG/ghgrabber)

`2010–2016`
**AtomicRMI:** Pessimistic distributed transactional memory system over Java RMI  
[`🔗`](https://github.com/kondziu/AtomicRMI)
Implementation of pessimistic transactional concurrency control for Java RMI.
RMI objects are instrumented to 
The algorithm assigns versions to shared objects and uses them to guide how transactions lock and release them.
It uses upper bounds on the number of accesses of an object within transactions to release locks early, if this is safe. 
It also uses local buffers to defer the need to synchronize transactions in specific situations.  
My contribution: Optimizations to the original algorithm, most of the implementation.

**GrittyScripts**


## Teaching

`2019-2021`
Runtime systems (NI-RUN)
Czech Technical University in Prague
https://courses.fit.cvut.cz/NI-RUN/

`2017`
Expeditions in Data Science (DS6050)
with Jan Vitek
http://janvitek.org/events/NEU/6050/

`2017`
Parallel Data Processing in MapReduce (DS6240)
with Jan Vitek
Northeastern University
http://janvitek.org/pdpmr/f17/

`2014–2016`
Safe programming methods (functional programming)
with Paweł T. Wojciechowski
Poznań University of Technology
http://www.cs.put.poznan.pl/pawelw/mbp/
http://www.cs.put.poznan.pl/ksiek/fp/fp.html

`2012–2013, 2016`
Networks
with 
Poznań University of Technology



`2009–2016`
Operating systems 
Poznań University of Technology

`2009–2017`
Basic IT
Poznań University of Technology (for Poznań University of Medical Sciences students)
[`🔗`](http://www.cs.put.poznan.pl/ksiek/pi/pi.html)







## Supervised theses

`submitted 2021`
Nilay Baranwal. *Structured printing framework.*  
Bachelor thesis at Czech Technical University in Prague.  

`2021`
Jan Jindráček. *Usability improvements to JavaScript/ECMAScript.*  
Master thesis at Czech Technical University in Prague.  

`2016`
Kamil Kozubal, Jakub Cieślak. *Hummy—An implementation of distributed transactional memory focused on performance.*  
Engineering thesis at Poznań University of Technology.  
Assistant supervisor under Paweł T. Wojciechowski. 

`2015`
Martin Witczak. *Atomic Café—A distributed multimedia playback system.*  
Master thesis at Poznań University of Technology.  
Assistant supervisor under Paweł T. Wojciechowski.

`2015`
Jan Baranowski. *Benchmarks for evaluating distributed transactional memory.*  
Master thesis at Poznań University of Technology.  
Assistant supervisor under Paweł T. Wojciechowski.

## Languages

`Proficient` English, Polish  
`Beginner` Czech, French  

## Programming languages

`Up-to-date` Rust, C, R, Bash, AWK, LaTeX  
`Rusty` Scala, Java, Python, OCaml  

## Extracurricular work

I was involved in a student club. I did work on PIWO. I organized workshops and lectures.

I organized remedial scala classes.

Library

## Hobbies

Inking  
Taking overexposed photos  
Uncool musical instruments  
Bad sci-fi  
Making lesson plans to explain declension  
Puns and haiku  
Kendo for one summer  

<hr/>

## Papers

Paweł Kobyliński, Konrad Siek, Jan Baranowski, Paweł T. Wojciechowski.  
**Helenos: A Realistic Benchmark for Distributed Transactional Memory**.  
Journal of Software: Practice and Experience.
Volume 48, issue 3.  
March 2018.    
[[Wiley](http://onlinelibrary.wiley.com/doi/10.1002/spe.2548/full)]

<hr/>

Konrad Siek, Paweł T. Wojciechowski.  
**Proving Opacity of Transactional Memory with Early Release**.  
Foundations of Computing and Decision Sciences.
Volume 40, issue 4.
December 2015.  
[[De Gruyter](http://www.degruyter.com/view/j/fcds.2015.40.issue-4/fcds-2015-0018/fcds-2015-0018.xml?format=INT>)]

<hr/>

Konrad Siek, Paweł T. Wojciechowski.  
**Atomic RMI: a Distributed Transactional Memory Framework**.  
In proceedings of HLPP 2014: the 7th International Symposium on High-level Parallel Programming and Applications.
International Journal of Parallel Programming, Volume 44, Issue 3, pp 598-619. April 2015.  
[[Springer](http://link.springer.com/article/10.1007/s10766-015-0361-x>)]

<hr/>

Konrad Siek, Paweł T. Wojciechowski.  
**A Formal Design of a Tool for Static Analysis of Upper Bounds on Object Calls in Java**.  
In Proceedings of FMICS 2012: the 17th International Workshop on Formal Methods for Industrial Critical Systems (co-located with FM 2012).  
Lecture Notes in Computer Science Volume 7437, pp 192–206. August 2012.  
[[Springer](https://link.springer.com/chapter/10.1007/978-3-642-32469-7_13), [PDF](pub/fmics12.pdf)]

<br/>

## Short papers

Paweł T. Wojciechowski, Konrad Siek.  
**Atomic RMI 2: Distributed Transactions for Java**.  
In Proceedings of AGERE'16: the 6th International Workshop on Programming Based on Actors, Agents, and Decentralized Control. October 2016.  
[[abstract](https://2016.splashcon.org/event/agere2016-atomic-rmi-2-distributed-transactions-for-java)]

Konrad Siek, Paweł T. Wojciechowski.   
**Brief Announcement: Relaxing Opacity in Pessimistic Transactional Memory**.  
In Proceedings of DISC'14: the 28th International Symposium on Distributed Computing.
October 2014.  
[[PDF](pub/disc14.pdf)]

<hr/>

Konrad Siek, Paweł T. Wojciechowski.  
**Zen and the Art of Concurrency Control: An Exploration of TM Safety Property Space with Early Release in Mind**.  
In Proceedings of WTTM'14: the 6th Workshop on the Theory of Transactional Memory. 
July 2014.  
[[PDF](pub/wttm14.pdf)]

<hr/>

Paweł T. Wojciechowski, Konrad Siek.  
**Having Your Cake and Eating it Too: Combining Strong and Eventual Consistency**.  
In Proceedings of PaPEC 2014: the 1st Workshop on the Principles and Practice of Eventual Consistency. April 2014.  
[[PDF](pub/papec14.pdf)]

<hr/>

Konrad Siek, Paweł T. Wojciechowski.  
**Towards a Fully-Articulated Pessimistic Distributed Transactional Memory** (brief announcement).  
In Proceedings of SPAA 2013: the 25th ACM Symposium on Parallelism in Algorithms and Architectures.
July 2013.  
[[PDF](pub/spaa13.pdf)]

<hr/>

Paweł T. Wojciechowski, Konrad Siek.  
**Rollbacks in Pessimistic Distributed TM**.  
SRDC'13: TRANSFORM Summer School on Research Directions in Distributed Computing. June 2013.  
[[abstract](srdc13-abstract.pdf)]

<hr/>

Paweł T. Wojciechowski, Konrad Siek.  
**Transaction Concurrency Control via Dynamic Scheduling Based on Static Analysis** (extended abstract).  
In Proceedings of WTM 2012: Euro-TM Workshop on Transactional Memory (co-located with ACM SIGOPS EuroSys 2012). April 2012.  
[[abstract](http://www.eurotm.org/action-meetings/wtm2012/program/abstracts#Wojciechowski)]

<hr/>

Konrad Siek, Paweł T. Wojciechowski.  
**Statically Computing Upper Bounds on Object Calls for Pessimistic Concurrency Control** (extended abstract).  
In Proceedings of EC :math:`^2` 2010: Workshop on Exploiting Concurrency Efficiently and Correctly (co-located with CAV 2010). July 2010.  
[[PDF](pub/sw10.pdf)]

<hr/>

## Technical reports

Jan Baranowski, Konrad Siek, Paweł T. Wojciechowski.  
**Analiza Programów Wzorcowych dla Rozproszonej Pamieci Transakcyjnej**.  
Raport RB-3/15. Instytut Informatyki Politechniki Poznańskiej.  
[[PDF](pub/rb-3-15.pdf)]

<hr/>

Martin Witczak, Konrad Siek.  
**Rozproszony System Zarządzania Odtwarzaniem Mediów w Oparciu o Pesymistyczną Pamięć Transakcyjną**.  
Raport RB-4/15. Instytut Informatyki Politechniki Poznańskiej.  
[[PDF](pub/rb-4-15.pdf)]