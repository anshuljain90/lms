i am a working professional into software develpment, desination is AI architect. there are many friends of mine and couleages asked me to teach them generative ai, agentic ai etc...

so i have started a batch, now its a mix of people from different domains (development, QA, networking,  finance and security) with different level of experience... 

now i wnt to teach them... for that i need to prepare the content and resources... for that iwant to use either chat gpt or claude or there api. 

so i want a full fleged web app which is responsive and can wrapped as a mobile app via webview...

in that i / admin should be able to  
- add/invite user by entering email along with some infor (optional) about them
- create courses and batches for the courses
- map users to batches
- approve request of users to register for a batch and have option to add remark and in pahse 2 option to add attchments like receipt of payment
- remove them from batch
- plan for any courses' content and other thing just like normal chatgept and claude web ui,  but via LLM api call
- can add relevant usefull info from the chat with LLM to the resources of that course of any particular batch of any course. 
- Add any material to the course or batch manually
- add task in any on going batch and have an option to add them as task to the course as well. the idea is based upon whats being taught the course content and task/assignments may evolve and may be to a batch i gave any task but might not be releavnt for other batches so i will keep that task  just for that batch
- all the tasks given should be visible to all students
- there could be tasks that are assigned to specific students in a batch
- every task should have general configurable fields  like deadline etc
- Also there could be tasks which are mandatory to complete before next session, such taks will have deadline atleast 1 hr before the class start and it will be mandatory for all students to appear for online AI assessment for that task and AI / LLM will ask questions based upon the task description and learning objectives set by instructor and then score the student.. here we will integrate LLM API also questions can be subjective like describe your understanding about LLMs,… them our AI assessor will judge the user and have a discussion in chat like human to evaluate them, also questions can be objective… 
- so when tasks are created our AI engine will create a pool of questions with different difficulty  and ask users those questions in random order and  do ask all questions to all users, just a subset… and questions can be mew style or text input style..and assessment should be chat style like any user chatting with tool like claude.ai or chat got and user input should be allowed only for text input questions otherwise give questions with clickable options.
- should see who all have completed how many tasks 


user can 
- register them selves or invited by me/ admin, 
- login, 
- reset password,
- we can have simple login using google in phase 2 where in we will use keycloak
- user can update their profile, verify email address. showa verified flag against email, if join by admin invite then verified by default because admin invite will be sent via email anduse will join via emailed link, 
- see list of courses and ongoing/ upcoming batch
- send admin request to enroll in any batch along with a option to enter optional text
- see assigned tasks  with there time left
- appear for AI assessment for the given task(s)


Above are the features I could think of.. apart from these add common features which are there in any LMS. 

I want to create a personalised learning platform where instructor can pay attention to each student from diff background and experience level because thats what the real chakllange with the students… same content for diff students…

Also I want you not to assume things ask me if you need clarity about the features or if you have any doubt…

Make a list of features… break them into phases so that we can test each phase if its working…

Use mono repo, create a single docker compose file using which I should be able to rest it locally and run on a server may be for testing…

For local testing make it configurable to use any LLM PAI or use ollama…

Use frameworks and libraries which keep my code any api provider agnostic…

If you want to create 2 different AI agents one for course planning and one for assessment thats fine or is you simply want to integrate apis that also okay by me… just that if you plan to create AI agents then they must have well defined goals and have proper guardrails in place so that they can not be exploited by prompt injection or by any other means..

Aslo we MUST use portkey for LLM calls and have a backup LLM configured and we should have proper observability in place, also there should be  tracking and limits on tokens consumed by user during assessment, instructors during planing, this should be visible in a separate dashboard clearly


So there are several aspects like 
- finalising list of features
- designing the BE and DB schema and creating API contract for the FE & BE
- develop refreshing, responsive frontend
- develop BE
- infra side: create docker images, docker compose, gobervibility stack using open search or ELK, kibana , grafana, portkey, langsmith
- test: create test for each phase and execute tests before moving onto development of next phase
- create detailed CLAUDE.md first… then create different phase_<number>.md file in /docs folder 
- then start the implementation and testing phase wise and proceeding to next phase
- user multiple parallel agents wherever it make sense
