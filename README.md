DAG Build Plugin for Jenkins
=====================

**Preface**  
DAG stands for Directed Acyclic Graph . This term is commonly used in software development when discussing dependency trees.  In the context of Jenkins, "DAG Build" provides the necessary logic to calculate a DAG out of the existing Jenkins Dependency Graph defined in Jenkins projects for each build. 

__Update: 2/16/2019__
I have just been informed that all of the dependencygraph api surface that exists in the Jenkins documentation DOES NOT APPLY to pipelines. I am now exploring the API surface that does apply to those, which exists in: `org.jenkinsci.plugins.workflow.job`

Specifically, `WorkflowJob` run `WorkflowRun`

After speaking with mods in #jenkins, it seems the area to explore is hooking into the each `upstream` job, at the `node` event which actually triggers downstream jobs. If we can capture and drop the event there, it would actually be preferrable to the original approach as it would prevent builds from even being spawned. However, it does require a bit of re-working of the logic.

At the end of each `build`, enumerate the list of `downstreams` which are about to be triggered. This includes jobs listed as `downstreams()` in the current job trigger definition, as well as jobs which list the current job in an `upstream()` trigger definition.

For each `downstream`, enumerate the list of upstreams "on behalf of that job". And then, perform the same algorithm previously discussed, but again "on behalf of" each downstream job. This job will then use the result of the algorithm to make a final decision will be whether or not to actually send the trigger to each downstream job.


**DAG Build Feature Summary**   
The DAG Build plugin builds adds automatic enforcement of a correct, coherent, ordered, and linear execution of a dependency graph, with support for complex fork and join scenarios. Notably, it works with both "Upstream" and "Downstream" build triggers. Historically, other strategies and mechanisms have enforced this type of deliberate, coordinated build logic within Jenkins, however they all have dramatically different designs, and thus have different characteristics. 

**Motivation**  
This plugin is particularly relevant for Jenkins deployments with complex build dependency graphs featuring "diamond dependency" scenarios.  Abstractly, a diamond dependency is one in which: 

    D depends on both C and B, which both depend on A

The challenge posed by diamond dependencies (whether using upstream or downstream triggers) is that intermediate dependencies (B and C in the example), can complete at different times, and will both trigger builds on any shared "Downstreams".  This is not only redundant and inefficient, it is fundamentally flawed because theoretically, both B and C must finish before D should be able to start.  So: 

    A -> trigger -> B
    A -> trigger -> C

    B -> trigger -> D -> D fails because C has not rebuilt with changes yet 

    C -> trigger -> D -> D succeeds

In this simple example, the result was 1 failed build.  In larger tree's, diamond dependency scenarios can set off very long chains of unnecessary failures and rebuilds which is bad in many different ways. Also, if `C` fails, then we never wanted `D` to run in the first place, and it's now sitting in a failed state until we fix `C`. 

**Jenkins Upstream/Downstrea Triggers**  
This plugin is also designed specifically for use with Jenkins declarative pipelines, and the advent of the "Uptream" and "Downstream" dependency declaration system.  In particular, the "Upstream" declarations are highly advantageous in large complicated build dependency graphs. In many cases, it is preferable to a "Downstream" declaration model, or monolithic pipeline definition, although "Downstream" model benefits just as well.  

**Strategy**  
The business logic of this plugin is surprisingly simple. The cornerstone strategy of DAG Build is to perform all the logic and calculation whenever a build is about to start, and that build was triggered by an "Upstream" cause. At that time, DAG Build is able to easily determine the specific "Root Upstream Build" which spawned the current "Build Tree", calculate all of the "Direct Upstreams" of the "Current Project" which will be built as part of that Build Tree, and verify that all of those Upstreams have been built successfully.  If they have not, the build is discarded and not run because it is not yet the correct time.  It's guaranteed that it will receive one or more additional "Upstream" causes from the remaining "Direct Upstreams" when they finally do build. 

In the diamond dependency example, the relevant change will be:

    A -> trigger -> B
    A -> trigger -> C

    B -> trigger -> D -> DAG Build Logic -> No "build" of D is created (Item.Task deleted)

    C -> trigger -> D -> DAG Build Logic -> A "build" of D is created and succeeds as normal


**Fundamental Benefits**  
Compared to other approaches, this solution provides the unique and favorable characteristics: 
- It supports declarative pipelines
- It's based entirely on native upstream/downstream declarations
- No new variables need to be passed between jobs
- It is stateless, no new state needs to be managed by Jenkins
- It introduces no new scaleability challenges

**Differences from Fork/Join strategy**  
Approaches which feature a Fork/Join strategy to dealing with diamond dependencies are fundamentally different in that they requires declarations of multiple relationships between upstream and downstream Jobs in some single location. Typically, metajobs are created to orchestrate a whole pipeline and manage these fork/join occasions.  This forces several design decisions that would otherwise be unnecessary and undesirable, and presents scalability challenges as wel.  
 
**Future**  
The plugin must exist as a plugin for quite some time while it is proven, adopted, and refined.  However, long-term, due to various characteristics of this plugin, there is a strong argument for it's logic to simply be included into Jenkins.  In theory, it could be refactored into a new set of options surrounding the existing upstream and downstream system. Reasons for making it native feature inlcude: 
- Computing a DAG from a dependency graph is a fundamental dependency concept
- Jenkins already does complete graph computations for each job
- Compatibility with pipelines
- Simplicity of domain logic
- Broadness of use-cases
- Number of advantages compared to existing solutions

There are also significant improvements which could be made if it was part of the native pipeline system.  Crucially, there is one current negative side effect of this plugin, which is that the "discarded" tasks have already been assigned job numbers by the time they are deleted. Thus job logs currently show numerical gaps for each deleted job. For example, a log might now show job numbers such as 4, 5, 9, 10. This could easily be circumvented by performing the logic at the moment job triggers are executed (at the end of the upstream job). The algorithm would be slightly modified, but functionally equivalent and much more efficient.

**Implementation**  
The implementation is still in progress, but extremely simple. `DagBuildListener` class implements `QueueListener` which implements `onEnterWaiting` method which receives the following parameter from jenkins: `Queue.WaitingItem wi`. In that function, it gets the root upstream cause here: 

```
Optional<Cause.UpstreamCause> rootUpstreamCause = wi
    .getCauses()
    .stream()
    .filter(Cause.UpstreamCause.class::isInstance)
    .map(Cause.UpstreamCause.class::cast)
    .reduce((a, b) -> b);
```

Two more things are left to do to implement this logic: 

1.  We need to get a handle to the `AbstractBuild` instance for that root upstream cause, and use it's `getDownstreams()` identify all jobs that will be spawned as a result. 
1.  We need to determine which of those builds represent direct upstreams of the current build.
1.  We need to determine if any of those direct upstream builds have not started or completed. 

If any of the direct upstreams builds haven't run and completed successfully as a result of the root upstream, we delete the current `WaitingItem` task. If all upstreams are passing successfully, we know we'll eventually get another trigger for each direct upstream.  Fundamentally, we only want to run when the very last direct upstream completes successfully, and only if all upstreams have completed successfully.

The current holdup is that the `Cause`/`Job`/`Build`/`DependencyGraph` object heirarchy is new to us, so it's taking time to figure out the optimal way to traverse/obtain the information. For someone familiar, it could be very simple.

**Summary**  
We are interested in feedback from anyone who has any level of understanding of this problem domain. Please post comments here on this gist. We'll create a github repository with the code soon, and move discussion there if the POC is successful. 