<p><a target="_blank" href="https://app.eraser.io/workspace/GuU9IoDGx5EKPUBCENl4" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>





- imported a lot of databases to test the joins and all
- made the select query capable of handling multi table permissions
- made the question associated with the multiple tables
- send success /table/execute
- r&d for the [﻿sharefiles.com](https://sharefiles.com/) we can do craete users they are known as create client users;


```
//Create payload for new Folder
var folderModel = 
{
    "Name": "Client files",
    "Description": "Folder to put this client's files"
}
 
//Make request to create new Folder
var createdFolder = MakeRequest("POST", "/sf/v3/Items/Folder", folderModel);
 
//Create payload for new Client User
var userModel = 
{
    "Email":"client@test.com",
    "FirstName": "Name",
    "LastName": " Last",
    "Company": "Company",
    "Password":"password",
    "Preferences":
    {
        "CanResetPassword":true,
        "CanViewMySettings":false
    }
}
 
//Make request to create new Client User. Parameter notify=true means the user will get an activation email
var createdUser = MakeRequest("POST", "/sf/v3/Users?notify=true", userModel);
 
//Create payload for new AccessControl object. These permissions will apply to the user on the new Folder
var accessControlModel =
{
    "Principal":{"Id": createdUser["Id"]},
    "CanUpload": false,
    "CanDownload": true,
    "CanView": true,
    "CanDelete": false,
    "CanManagePermissions": false
}
 
//Create new AccessControl, effectively sharing the new Folder with the created User
var accessControl = MakeRequest("POST", "/sf/v3/Items(" + createdFolder["id"] + ")/AccessControls", accessControlModel);
```




<!--- Eraser file: https://app.eraser.io/workspace/GuU9IoDGx5EKPUBCENl4 --->