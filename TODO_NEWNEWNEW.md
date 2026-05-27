Important:
Currently, the system doesn't let you rewind to a phase and rework from there. We should be able to not just replay, but also set the phase.
a real example is that I just tried making a test.md file, with all phases having "read-only" perms, that obviously failed, and the orchestrator
encouraged to give executor write access and run it again. How do I do it tho? There's no visible way to change the executor's agent write permissions
and re-run from that point. do we want this behavior?

Guides should have a target "Complexity" level. Since some tasks are "too easy" to accomplish, it will be an overkill
and waste of resources to run a complicated guide with multiple (Excessive) phases. a complicated flow. we should have
an estimation of the complexity of a given task, and set guides to meter a reasonable complexity level, so it will suggest that
"This flow might be too much for this task, use a more simple flow"
we have something like this in effort level choosing for our models, this may be relevant!


"blocked" phase on runs is awful. first, I hate how to access runs you have to go through all runs . 
Agents and providers can be probably unified as a single thing? Should we call it "Agents" or providers?

I feel like our naming makes sense now. We have a Crew (bunch of provider / agent), we have a flow (based on a guide? should it be just a flow or a guide?)
we then have task and run? whats the difference? shouldn't be tasks, and we can see a run which is basically the running of the crew ? but then where's the naming of
the orchestrator? What is it ? whats the supervisor? So , lets rethink about the naming, we want unified as possible, and least as possible to keep things clear and concise.

Windows not supported?