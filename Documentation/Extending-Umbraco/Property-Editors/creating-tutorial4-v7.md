#Tutorial - Adding server-side data to a property editor

##Overview
In this tutorial we will add a server-side API controller, which will query a custom table in the Umbraco database, and then return the data to a simple angular controller + view.

The end result will be a person-list, populated from a custom table, when clicked it will store the ID of the selected person.

##Setup the database
First thing we need is some data, below is a simple SQL Script for create a `people` table with some random data in. You could also use [http://generatedata.com] for larger amounts of data:

	CREATE TABLE people (
	    id INTEGER NOT NULL IDENTITY(1, 1),
	    name VARCHAR(255) NULL,
	    town VARCHAR(255) NULL,
	    country VARCHAR(100) NULL,
	    PRIMARY KEY (id)
	);
	GO

	INSERT INTO people(name,town,country) VALUES('Myles A. Pearson','Tailles','United Kingdom');
	INSERT INTO people(name,town,country) VALUES('Cora Y. Kelly','Froidchapelle','Latvia');
	INSERT INTO people(name,town,country) VALUES('Brooke Baxter','Mogi das Cruzes','Grenada');
	INSERT INTO people(name,town,country) VALUES('Illiana T. Strong','Bevel','Bhutan');
	INSERT INTO people(name,town,country) VALUES('Kaye Frederick','Rothesay','Turkmenistan');
	INSERT INTO people(name,town,country) VALUES('Erasmus Camacho','Sint-Pieters-Kapelle','Saint Vincent and The Grenadines');
	INSERT INTO people(name,town,country) VALUES('Aimee Sampson','Hawera','Antigua and Barbuda');


##Setup ApiController routes
Next we need to defined a `ApiController` to expose a server-side route which our application will use to fetch the data.

For this, we will create a file at: `/App_Code/PersonApiController.cs` It must be in app_code since we want our app to compile it on start, alternatively, you can just add it to a normal .net project and compile into a dll as normal.

In the PersonApiController.cs file, add: 

	using System;
	using System.Collections.Generic;
	using System.Linq;
	using System.Web;

	using Umbraco.Web.WebApi;
	using Umbraco.Web.Editors;
	using Umbraco.Core.Persistence;

	namespace My.Controllers
	{
	    [Umbraco.Web.Mvc.PluginController("My")]
	    public class PersonApiController : UmbracoAuthorizedJsonController
	    {
	        //we will add a method here later
	    }
	}

This is a very basic Api controller which inherits from `UmbracoAuthorizedJsonController` this specific class will only return json data, and only to requests which are authorized to access the backoffice

##Setup the GetAll() method
Now that we have a controller, we need to create a method, which can return a collection of people, which our editor will use. 

So first of all, we add a `Person` class to the `My.Controllers` namespace:

	public class Person
	{
	    public int Id { get; set; }
	    public string Name { get; set; }
	    public string Town { get; set; }
	    public string Country { get; set; }
	}

We will use this class to map our table data to an c# class, which we can return as json later. 

Now we need the `GetAll()` method which returns a collection of people, insert this inside the PersonApiController class:

	public IEnumerable<Person> GetAll()
	{
		
	}

Inside the GetAll() method, we now write a bit of code, that connects to the database, creates a query and returns the data, mapped to the `Person` class above: 

	//get the database
	var db = UmbracoContext.Application.DatabaseContext.Database;
	//build a query to select everything the people table
	var query = new Sql().Select("*").From("people");
	//fetch data from DB with the query and map to Person object
	return db.Fetch<Person>(query);

We are now done with the server-side of things, with the file saved in app_code you can now open the Url: /umbraco/backoffice/My/PersonApi/GetAll

This will return our json code.

##Create a Person Resource 
Now that we have the server-side in place, and a Url to call, we will setup a service to retrieve our data. As an Umbraco specific convention, we call these services a *resource, so we always have an indication what services fetch data from the DB.

Create a new file as `person.resource.js` and add: 

	//adds the resource to umbraco.resources module:
	angular.module('umbraco.resources').factory('personResource', 
		function($q, $http) {
		    //the factory object returned
		    return {
		        //this cals the Api Controller we setup earlier
		        getAll: function () {
		            return  $http.get("backoffice/My/PersonApi/GetAll");
		        }
		    };
		}
	); 

This uses the standard angular factory pattern, so we can now inject this into any of our controllers under the name `personResource`.

the getAll method just returns a $http.get call, which handles calling the url, and will return the data when its ready.

##Create the view and controller
We will now finally setup a new view and controller, which follows previous tutorials, so have refer to those for more details: 

####the view:

	<div ng-controller="My.PersonPickerController">
		<ul>
			<li ng-repeat="person in people">
				<a href ng-click="model.value = person.Name">{{person.Name}}</a>
			</li>
		</ul>
	</div>

####The controller:
	
	angular.module("umbraco")
		.controller("My.PersonPickerController", function($scope, personResource){
			personResource.getAll().then(function(response){
				$scope.people = response.data;
			});
		});

##The flow
So with all these bits in place, all you need to do is register the property editor in a package.manifest - have a look at the first tutorial in this series. You will need to tell the package to load both your personpicker.controller.js and the person.resource.js file on app start.

With this, the entire flow is: 

1. the view renders a list of people with a controller
2. the controller asks the personResource for data
3. the personResource returns a promise and asks the my/PersonAPI api controller
4. The apicontroller queries the database, which returns the data as strongly typed Person objects
5. the api controller returns those `Person` objects as json to the resource
6. the resource resolve the promise
7. the controller populates the view

Easy huh? - honestly tho, there is a good amount of things to keep track of, but each component is tiny and flexible. 

##Wrap-up
The important part of the above is the way you create an `ApiController` call the database for your own data, and finally expose the data to angular as a service using $http.

For simplicity, you could also have skipped the service part, and just called $http directly in your controller, but by having your data in a service, it becomes a reusable resource for your entire application.
