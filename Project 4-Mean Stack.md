## MEAN STACK DEPLOYMENT TO UBUNTU IN AWS
### Task- Implement a simple Book Register web form using MEAN stack.
#### Steps
1. ##### Install Nodejs
  * Provision Ubuntu 20.4 instance in AWS
  * Connect to the instance through an SSH client.
  * Once in the terminal, update Ubuntu using this command: `sudo apt update`
  * Next, upgrade Ubuntu with `sudo apt upgrade`
  * Add certificates: 
```
sudo apt -y install curl dirmngr apt-transport-https lsb-release ca-certificates

curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
```
![aaaa](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/a686c139-1355-45c0-b365-35219a5c0ae3)

  * Next we install nodejs with this: `sudo apt install -y nodejs`
  
 ![bbb](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/5eb29139-3fd8-42dd-8cf1-1b6a0006decc)
  
2. ##### Install MongoDB
  * First we add our MongoDB key server with: `sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6`
  * Add repository: `echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu trusty/mongodb-org/3.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.4.list`
  * Install MongoDB with the following comand: `sudo apt install -y mongodb`

  ![hsjkhf](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/bc77a82a-f840-4215-8316-c52c29765b9e)

  
  * Verify Server is up and running: `sudo systemctl status mongodb`

  ![dkaksds](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/5f44e020-0576-4f71-b5d8-bc777bdfa7ae)

  
  * Install npm – Node package manager: `sudo apt install -y npm`
  * Next we install body-parser package to help with processing JSON files passed in requests to the server. Use the following command: `sudo npm install body-parser`
  
 ![eeeee](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/89f587cb-aa06-4fb7-879c-3659f901a261)
 
  
  * Next we create the **Books** directory and navigate into it with the following command: `mkdir Books && cd Books` 
  * Inside the Books directory initialize npm project and add a file to it with the following command: `npm init` Then add **sever.js** file with: `vi server.js`
  * In the server.js file, paste the following code:
  ```
  var express = require('express');
var bodyParser = require('body-parser');
var app = express();
app.use(express.static(__dirname + '/public'));
app.use(bodyParser.json());
require('./apps/routes')(app);
app.set('port', 3300);
app.listen(app.get('port'), function() {
    console.log('Server up: http://localhost:' + app.get('port'));
});
```
![fffff](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/6cf5fbca-c4a6-48ea-9f4a-7aac431f1e4f)

3. ##### Install Express and set up routes to the server
 * Express will be used to pass book information to and from our MongoDB database and Mongoose will be used to establish a schema for the database to store data of our book register. To begin installation, type: `sudo npm install express mongoose` and enter.
 
 ![gggg](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/6a2fa29d-b882-4907-bed0-b1353958c428)
 
 
 * while in **Books** folder, create a directory named **apps** and navigate into it with: `mkdir apps && cd apps`
 * Inside **apps**, create a file named routes.js with: `vi routes.js`
 * Copy and paste the code below into routes.js
 ```
module.exports = function(app) {
  app.get('/book', function(req, res) {
    Book.find({}, function(err, result) {
      if ( err ) throw err;
      res.json(result);
    });
  }); 
  app.post('/book', function(req, res) {
    var book = new Book( {
      name:req.body.name,
      isbn:req.body.isbn,
      author:req.body.author,
      pages:req.body.pages
    });
    book.save(function(err, result) {
      if ( err ) throw err;
      res.json( {
        message:"Successfully added book",
        book:result
      });
    });
  });
  app.delete("/book/:isbn", function(req, res) {
    Book.findOneAndRemove(req.query, function(err, result) {
      if ( err ) throw err;
      res.json( {
        message: "Successfully deleted the book",
        book: result
      });
    });
  });
  var path = require('path');
  app.get('*', function(req, res) {
    res.sendfile(path.join(__dirname + '/public', 'index.html'));
  });
};
 ```
 ![hhhh](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/ca77a959-292a-4be6-9a75-ef26e335ffca)
 
 * Also create a folder named **models** in the **apps** folder, then navigate into it: `mkdir models && cd models`
 * Inside **models**, create a file named **book.js** with: `vi book.js`
 * Copy and paste the code below into ‘book.js’
```
var mongoose = require('mongoose');
var dbHost = 'mongodb://localhost:27017/test';
mongoose.connect(dbHost);
mongoose.connection;
mongoose.set('debug', true);
var bookSchema = mongoose.Schema( {
  name: String,
  isbn: {type: String, index: true},
  author: String,
  pages: Number
});
var Book = mongoose.model('Book', bookSchema);
module.exports = mongoose.model('Book', bookSchema);
```
![iiiiii](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/7fd24e89-2e05-44eb-9035-1b5e328bee73)

4. ##### Access the routes with AngularJS
 * The final step would be using AngularJS to connect our web page with Express and perform actions on our book register.
 * Navigate back to **Books** directory using: `cd ../..`
 * Now create a folder named **public** and move into it: `mkdir public && cd public`
 * Add a file named **script.js**: `vi script.js`
 * And copy and paste the following code:
 ```
 var app = angular.module('myApp', []);
app.controller('myCtrl', function($scope, $http) {
  $http( {
    method: 'GET',
    url: '/book'
  }).then(function successCallback(response) {
    $scope.books = response.data;
  }, function errorCallback(response) {
    console.log('Error: ' + response);
  });
  $scope.del_book = function(book) {
    $http( {
      method: 'DELETE',
      url: '/book/:isbn',
      params: {'isbn': book.isbn}
    }).then(function successCallback(response) {
      console.log(response);
    }, function errorCallback(response) {
      console.log('Error: ' + response);
    });
  };
  $scope.add_book = function() {
    var body = '{ "name": "' + $scope.Name + 
    '", "isbn": "' + $scope.Isbn +
    '", "author": "' + $scope.Author + 
    '", "pages": "' + $scope.Pages + '" }';
    $http({
      method: 'POST',
      url: '/book',
      data: body
    }).then(function successCallback(response) {
      console.log(response);
    }, function errorCallback(response) {
      console.log('Error: ' + response);
    });
  };
});
```
![jjjj](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/9f6cda08-b285-453e-9e1a-4f0dff56c11e)


* Also in **public** folder, create a file named **index.html**: `vi index.html`
* And and paste the foloowing html code below into it: 
```
<!doctype html>
<html ng-app="myApp" ng-controller="myCtrl">
  <head>
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.4/angular.min.js"></script>
    <script src="script.js"></script>
  </head>
  <body>
    <div>
      <table>
        <tr>
          <td>Name:</td>
          <td><input type="text" ng-model="Name"></td>
        </tr>
        <tr>
          <td>Isbn:</td>
          <td><input type="text" ng-model="Isbn"></td>
        </tr>
        <tr>
          <td>Author:</td>
          <td><input type="text" ng-model="Author"></td>
        </tr>
        <tr>
          <td>Pages:</td>
          <td><input type="number" ng-model="Pages"></td>
        </tr>
      </table>
      <button ng-click="add_book()">Add</button>
    </div>
    <hr>
    <div>
      <table>
        <tr>
          <th>Name</th>
          <th>Isbn</th>
          <th>Author</th>
          <th>Pages</th>

        </tr>
        <tr ng-repeat="book in books">
          <td>{{book.name}}</td>
          <td>{{book.isbn}}</td>
          <td>{{book.author}}</td>
          <td>{{book.pages}}</td>

          <td><input type="button" value="Delete" data-ng-click="del_book(book)"></td>
        </tr>
      </table>
    </div>
  </body>
</html>
```

* Then change the directory back up to **Books** using: `cd ..`
* Now, start the server by running this command: `node server.js` If all goes well server should be up and running and we can connect to it on port 3300.
* This however was not the case in my test, I kept on getting an error. see error in the picture below:
 
![llllllll](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/f07ee3d6-2a3e-4642-9fd7-3aa8e0a355cf)

* After several attempts at troubleshooting I figured out what the issue was. The issue was with my nodejs version. During the installation, I installed version 12 but for some reson version 12 won't work but keeps giving me errors when I attempt to start the sever.
* I solved this problem by upgrading my node version to version, 17.0.0
* After this I tried to start the server again and it ran successfully.

![Project4pix16](![mmmmm](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/a25f7740-7489-4b8e-8d91-7a59a92bed85)


* Next I accessed the HTML page over the internet via port 3300 using the public IP: `http://34.200.223.160:3300/`

![nnnnnnn](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/b63f514b-76aa-4154-8864-483f37ec9cfa)


* Finally I enter some data into the database and it reflected.

![oooooo](https://github.com/Suleiman223/DevOps-Projects/assets/116959775/6f3faf00-36d5-44f8-b6f5-53073957a4d1)



