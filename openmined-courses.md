# Learning platform offered by Openmined.org. 

## Overall architecture
- *Backend*: Firebase **clouding functions**
- *Frontend*: **NextJS** hosted on **firebase**
- *CMS*: Managing courses with **Sanity.io**
- *Other*:
	- **Stripe** for payment
	- **ChakraUI** for UI components
	- **Nx** for monorepo

Primary courses content is saved on cloud sanity server hosted ty sanity.io. Each user has limited access to courses depending on payment status.

## Courses
Each course is composed of lessons, projects. Each lesson can be a mixed markdown content of video/image/questions/plain text.

Each data entity is built as schema within sanity.


## User Levels
There are 3 User levels.
- admin: Manage courses and all users
- mentor: Manage project approval, rejection and mentoring.
- user: Actually study courses and submit project result.


## Some issues I had and how I solved it.

### Firebase permission layer.
We should define who can review whose project, who can view which courses etc. Everything should be all done firestore which has loose data structures.

Example permission rule.

      match /courses/{course} {
        function isCourseAssignedToMentor() {
          let mentorableCourses = getUserData(request.auth.uid).data.mentorable_courses;

          return mentorableCourses.hasAll([course]);
        }

        // Student can create
        allow create: if isSameUser(user);

        // A student can update their own course, and...
        // Mentor will need to be able to edit the submissions array of any user's course object with the "status" and "reviewed_at" time
        allow update: if isSameUser(user) || (isMentor(request.auth.uid) && hasAllAffectedKeys(['project']));

        // A student can read their own course, and...
        // Mentors should always be able to read any user's course object that they are allowed to review
        allow read: if isSameUser(user) || (isMentor(request.auth.uid) && isCourseAssignedToMentor());

        // Nobody can delete course progress
        allow delete: if false;

        // These are submissions of a student
        // But remember, this DOES NOT affect how a mentor would read this submission
        match /submissions/{submission} {
          // A student or the assigned mentor should be able to read a submission
          allow read: if isSameUser(user) || get(/databases/$(database)/documents/users/$(user)/courses/$(course)).data.allowed_mentors.values().hasAny([request.auth.uid]);

          // A student should be able to create a submission
          allow create: if isSameUser(user);

          // Only the assigned mentor should be able to update a submission
          allow update: if get(existingData().mentor).id == request.auth.uid;

          // Nobody can delete a submission
          allow delete: if false;
        }

        // A student should be allowed to leave feedback for some part of a course
        match /feedback/{conceptOrLesson} {
          allow read, write: if isSameUser(user)
        }
      }
    }
    
### How it shares code and types between FE and BE via NX.
![enter image description here](https://i.ibb.co/7jc9pR6/image.png)
