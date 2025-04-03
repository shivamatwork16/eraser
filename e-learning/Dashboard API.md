<p><a target="_blank" href="https://app.eraser.io/workspace/rswfv1NSbEO1KPFT35Yi" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

So we want to make an API which will show overall stats of our users and all it will have 

```javascript
[userCount] = db.Users.findAndCountAll({
  where :{
    isDeleted:false,role:"user"
  }
})

[questionCoutn] = db.QueryQuestion)

data = {
  users : 789,
  subadmins: 7,
  questions: 899,
  courses: 122
}

return res.json({success:true,
message: constanst.common.success
data: data,
})
```




<!--- Eraser file: https://app.eraser.io/workspace/rswfv1NSbEO1KPFT35Yi --->