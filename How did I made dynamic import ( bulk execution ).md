<p><a target="_blank" href="https://app.eraser.io/workspace/yNpK00qCio8rTALERwa6" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

## Preface
On march 18 I started working on the dynamic table generation API 

what we wanted to achieve from it 

- admin can create a new table
- admin can create dynamic schema with valid SQL types ( a lot combination we support many but nowhere near to all)
- admin can edit into it make changes in ANY column and remove add or whatever they want
so the above functionality was ready and we tested it works I have taken an approach of file system where in real time whenever a task for new model comes we create a new model and import it in our projects and synchronize it with the Sequalize.sync method, 

now the above method has flaws(such as sync method is not recommended for the production workload since it some data can be lost) still it works as expected



## why 
now that we had the dynamic data model ( aka table generation) functionality ready the admin would like to insert some data as well so one is that he can insert the data one by one

using some commands like 

```
INSERT INTO dynamically_generated_table (username, email)
VALUES ('Your Name', 'you@n3y.in');
```




<!--- Eraser file: https://app.eraser.io/workspace/yNpK00qCio8rTALERwa6 --->