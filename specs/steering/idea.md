Please develop a comprehensive spec to create a claude code plugin named 'ideate' which helps users plan, ideate, develop, and validate software.

This tool should feature an interactive workflow which interviews the user in order to take an idea and fully explore and refine it into an actionable spec.

I often use plan mode because it does a very good job creating specs, but i have noticed that it does have some limitations:
1) it creates a single body of work. Its not well suited for large apps and breaks work into phases which need constant review
2) its not very good at understanding intent and when to ask followup questions
3) once the plan is done, it doesn't iterate


The tool should have three phases of operation:
1) Plan
2) Execute
3) Review


# Plan
To start, it should interview the user in depth. when intiated, we should go back and forth where i start with an idea and the tool asks questions and refines an idea until the idea is fully fleshed out. This process should produce multiple artifacts:
1) A comprehensive, exaustive plan on what to create. This should be broken down to high granularity in a way that many agents can work cleanly in parallel. If reasonable, it should design around the features and limitations of claude code- it should utilize agent teams, subagents, git worktrees, etc. in order to be able to build in a parallel fashion. It should follow best practices of claude code as found on the documentation to accomplish the job as fast as possible. 
2) it should additionally create high level guiding pricipals about what to create. These should cover the "why" in addition to "what". They need to be sanitiy checks to see if the project being delivered is truely what the user wants.
3) it should determine any other resources which would be helpful to 'steer' the development of the project. 


As part of the interview: 
1) interview, it should spawn agents to research topic discussed. It should use any research skills available and should leverage agents who are specially designed to research and syntheise ideas.
2) it should prompt for how it should function. Running massively in parallel might be inefficient but could get the job done faster. It should interview on not just what to make, but how to make it. This also includes technologies, design decisions, target audiences, etc. The tool should determine what design constraints are imporant and cater its process around them

At the beginning of the session, the agent should ask where it should be saved. 
The interview itself should be stored as a steering document in a Q&A format for future reference. Its okay to pause while the researchers are reviewing topics brefore asking more questiongs. 
When the idea is fully explored, or the user has expressed intent to continute on, we should present the report and any outstanding concerns or issues which still remain. At this point, the user may indicate to start development in the next phase. 


# execute
When executing the plan, the leader should follow the strategy that the user specified for using agents. the leader should present regular status reports as the project is developed.  Upon completion of individual parts of the plan, reviewers should immediately be spawned to validate the work which was done. 


# review
Upon finishing the entire plan, the tool should additionally spawn an agent team of reviewers to evaluate the quality of the output. This will be more comprehensive than the individual checks during development. Each agent should specialize in different areas of evaluation (e.g. code quality, how well it adheres to the requests, any missing gaps, ideas for improvemets, missing requirements or design flaws). These are just examples but in general, we want agents to ensure that the code produced meets the goals we initially defined, as well as the specific plan. Any requirement misses of any kind should feed into subsequent refining phases. If gaps are unaddressed by the steering documents or high level guidance, it should prompt the user.

During the process, it should keep detailed notes of design challenges, important decisions, and any unaddressed issues or potential issues which could not be corrected.


# Agent languate and tone 
The tool should speak in a neutral tone and not provide validation or encouragement to the user. It should speak critically and honestly without trying to sugar coat anything. A bad idea is a bad idea and it is the tools responsibility to not provide validation, just honesty. It should also not qualify its language in ways which try to make it sound more genuine or sincere. Just neutral and to the point.

# Skills
The plugin has 4 skills- plan, execute, review, refine. As this is a plugin, they will be prefixed with 'ideate:' The first 3 skills are defined above. the forth, refine, is a special skill which performs plan in an existng codebase


----
If any part of these requirements are unclear, please ask clarifying questions until you can create a full spec. 