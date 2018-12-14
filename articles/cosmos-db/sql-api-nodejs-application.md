---
title: Build a Node.js web app using JavaScript SDK to manage Azure Cosmos DB SQL API data
description: This Node.js tutorial explores how to use Microsoft Azure Cosmos DB to store and access data from a Node.js Express web application hosted on Azure Websites.
services: cosmos-db
author: SnehaGunda

ms.service: cosmos-db
ms.component: cosmosdb-sql
ms.devlang: nodejs
ms.topic: tutorial
ms.date: 09/24/2018
ms.author: sngun

---
# Tutorial: Build a Node.js web app using JavaScript SDK to manage Azure Cosmos DB SQL API data

> [!div class="op_single_selector"]
> * [.NET](sql-api-dotnet-application.md)
> * [Java](sql-api-java-application.md)
> * [Node.js](sql-api-nodejs-application.md)
> * [Python](sql-api-python-application.md)
> * [Xamarin](mobile-apps-with-xamarin.md)
> 

This Node.js tutorial shows you how to store and access data from Azure Cosmos DB SQL API account by using a Node.js Express application that is hosted on Azure Websites. In this tutorial, you will build a web-based application (Todo app), that allows you to create, retrieve, and complete tasks. The tasks are stored as JSON documents in Azure Cosmos DB. 

This tutorial demonstrates how to create an Azure Cosmos DB SQL API account by using the Azure portal. You then build and run a web app that is built on the Node.js SDK to create database, container, and add items to the container. This tutorial uses JavaScript SDK version 2.0.

You can also get the completed sample from [GitHub][GitHub] and see the [Readme](https://github.com/Azure-Samples/documentdb-node-todo-app/blob/master/README.md) file for instructions on how to run the app.

This tutorial covers the following tasks:

> [!div class="checklist"]
> * Create an Azure Cosmos DB account
> * Create a new Node.js application
> * Connect the application to Azure Cosmos DB
> * Run and deploy the application to Azure

## <a name="_Toc395783176"></a>Prerequisites

Before following the instructions in this article, ensure
that you have the following resources:

* If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin. 

  [!INCLUDE [cosmos-db-emulator-docdb-api](../../includes/cosmos-db-emulator-docdb-api.md)]

* [Node.js][Node.js] version 6.10 or higher.
* [Express generator](https://www.expressjs.com/starter/generator.html) (you can install Express via `npm install express-generator -g`)
* Install [Git][Git] on your local workstation.

## <a name="_Toc395637761"></a>Step 1: Create an Azure Cosmos DB account
Let's start by creating an Azure Cosmos DB account. If you already have an account or if you are using the Azure Cosmos DB Emulator for this tutorial, you can skip to [Step 2: Create a new Node.js application](#_Toc395783178).

[!INCLUDE [cosmos-db-create-dbaccount](../../includes/cosmos-db-create-dbaccount.md)]

[!INCLUDE [cosmos-db-keys](../../includes/cosmos-db-keys.md)]

## <a name="_Toc395783178"></a>Step 2: Create a new Node.js application
Now let's learn to create a basic Hello World Node.js project using the [Express](https://expressjs.com/) framework.

1. Open your favorite terminal, such as the Node.js command prompt.

1. Navigate to the directory in which you'd like to store the new application.

1. Use the express generator to generate a new application called **todo**.

   ```bash
   express todo
   ```

1. Open the **todo** directory and install dependencies.

   ```bash
   cd todo
   npm install
   ```

1. Run the new application.

   ```bash
   npm start
   ```

1. You can view your new application by navigating your browser to [http://localhost:3000](http://localhost:3000).
   
   ![Learn Node.js - Screenshot of the Hello World application in a browser window](./media/sql-api-nodejs-application/cosmos-db-node-js-express.png)

 Stop the application by using CTRL+C in the terminal window and select **y** to terminate the batch job.

## <a name="_Toc395783179"></a>Step 3: Install the required modules

The **package.json** file is one of the files created in the root of the project. This file contains a list of additional modules that are required for your Node.js application. When you deploy this application to Azure, this file is used to determine which modules should be installed on Azure to support your application. Install two more packages for this tutorial.

1. Open the terminal, install the **async** module via npm.

   ```bash
   npm install async --save
   ```

2. Install the **@azure/cosmos** module via npm. 

   ```bash
   npm install @azure/cosmos
   ```

## <a name="_Toc395783180"></a>Step 4: Connect the Node.js application to Azure Cosmos DB
Now that you have completed the initial setup and configuration, next you will write code that is required by the todo application to communicate with Azure Cosmos DB.

### Create the model
1. At the root of your project directory, create a new directory named **models**  

2. In the **models** directory, create a new file named **taskDao.js**. This file contains code required to create the database, container and defines methods to read, update, create, and find tasks in Azure Cosmos DB. 

3. Copy the following code into the **taskDao.js** file

   ```nodejs
   // @ts-check
   const CosmosClient = require("@azure/cosmos").CosmosClient;
   const debug = require("debug")("todo:taskDao");
   class TaskDao {
     /**
      * Manages reading, adding, and updating Tasks in Cosmos DB
      * @param {CosmosClient} cosmosClient
      * @param {string} databaseId
      * @param {string} containerId
      */
     constructor(cosmosClient, databaseId, containerId) {
       this.client = cosmosClient;
       this.databaseId = databaseId;
       this.collectionId = containerId;

       this.database = null;
       this.container = null;
     }

     async init() {
       debug("Setting up the database...");
       const dbResponse = await this.client.databases.createIfNotExists({
         id: this.databaseId
       });
       this.database = dbResponse.database;
       debug("Setting up the database...done!");
       debug("Setting up the container...");
       const coResponse = await this.database.containers.createIfNotExists({
         id: this.collectionId
       });
       this.container = coResponse.container;
       debug("Setting up the container...done!");
     }

     async find(querySpec) {
       debug("Querying for items from the database");
       if (!this.container) {
         throw new Error("Collection is not initialized.");
       }
       const { result: results } = await this.container.items
        .query(querySpec)
        .toArray();
      return results;
    }

    async addItem(item) {
      debug("Adding an item to the database");
      item.date = Date.now();
      item.completed = false;
      const { body: doc } = await this.container.items.create(item);
      return doc;
    }

    async updateItem(itemId) {
      debug("Update an item in the database");
      const doc = await this.getItem(itemId);
      doc.completed = true;

      const { body: replaced } = await this.container.item(itemId).replace(doc);
      return replaced;
    }

    async getItem(itemId) {
      debug("Getting an item from the database");
      const { body } = await this.container.item(itemId).read();
      return body;
    }
  }

   module.exports = TaskDao;
   ```
4. Save and close the **taskDao.js** file.  

### Create the controller

1. In the **routes** directory of your project, create a new file named **tasklist.js**.  

2. Add the following code to **tasklist.js**. This code loads the CosmosClient and async modules, which are used by **tasklist.js**. This code also defines the **TaskList** class, which is passed as an instance of the **TaskDao** object we defined earlier:
   
   ```nodejs
   const TaskDao = require("../models/TaskDao");

   class TaskList {
     /**
      * Handles the various APIs for displaying and managing tasks
      * @param {TaskDao} taskDao
     */
    constructor(taskDao) {
    this.taskDao = taskDao;
    }
    async showTasks(req, res) {
      const querySpec = {
        query: "SELECT * FROM root r WHERE r.completed=@completed",
        parameters: [
          {
            name: "@completed",
            value: false
          }
        ]
      };

      const items = await this.taskDao.find(querySpec);
      res.render("index", {
        title: "My ToDo List ",
        tasks: items
      });
    }

    async addTask(req, res) {
      const item = req.body;

      await this.taskDao.addItem(item);
      res.redirect("/");
    }

    async completeTask(req, res) {
      const completedTasks = Object.keys(req.body);
      const tasks = [];

      completedTasks.forEach(task => {
        tasks.push(this.taskDao.updateItem(task));
      });

      await Promise.all(tasks);

      res.redirect("/");
    }
  }

  module.exports = TaskList;
   ```

3. Save and close the **tasklist.js** file.

### Add config.js

1. At the root of your project directory, create a new file named **config.js**. 

2. Add the following code to **config.js** file. This code defines configuration settings and values needed for our application.
   
   ```nodejs
   const config = {};

   config.host = process.env.HOST || "[the endpoint URI of your Azure Cosmos DB account]";
   config.authKey =
     process.env.AUTH_KEY || "[the PRIMARY KEY value of your Azure Cosmos DB account";
   config.databaseId = "ToDoList";
   config.containerId = "Items";

   if (config.host.includes("https://localhost:")) {
     console.log("Local environment detected");
     console.log("WARNING: Disabled checking of self-signed certs. Do not have this code in production.");
     process.env.NODE_TLS_REJECT_UNAUTHORIZED = "0";
     console.log(`Go to http://localhost:${process.env.PORT || '3000'} to try the sample.`);
   }

   module.exports = config;
   ```

3. In the **config.js** file, update the values of HOST and AUTH_KEY using the values found in the Keys page of your Azure Cosmos DB account on the [Microsoft Azure portal](https://portal.azure.com). 

4. Save and close the **config.js** file.

### Modify app.js

1. In the project directory, open the **app.js** file. This file was created earlier when the Express web application was created.  

2. Add the following code to the **app.js** file. This code defines the config file to be used, and loads the values into some variables that you will use in the next sections. 
   
   ```nodejs
   const CosmosClient = require("@azure/cosmos").CosmosClient;
   const config = require("./config");
   const TaskList = require("./routes/tasklist");
   const TaskDao = require("./models/taskDao");

   const express = require("express");
   const path = require("path");
   const logger = require("morgan");
   const cookieParser = require("cookie-parser");
   const bodyParser = require("body-parser");

   const app = express();

   // view engine setup
   app.set("views", path.join(__dirname, "views"));
   app.set("view engine", "jade");

   // uncomment after placing your favicon in /public
   //app.use(favicon(path.join(__dirname, 'public', 'favicon.ico')));
   app.use(logger("dev"));
   app.use(bodyParser.json());
   app.use(bodyParser.urlencoded({ extended: false }));
   app.use(cookieParser());
   app.use(express.static(path.join(__dirname, "public")));

   //Todo App:
   const cosmosClient = new CosmosClient({
     endpoint: config.host,
     auth: {
       masterKey: config.authKey
     }
   });
   const taskDao = new TaskDao(cosmosClient, config.databaseId, config.containerId);
   const taskList = new TaskList(taskDao);
   taskDao
     .init(err => {
       console.error(err);
     })
     .catch(err => {
       console.error(err);
       console.error("Shutting down because there was an error settinig up the database.");
       process.exit(1);
     });

   app.get("/", (req, res, next) => taskList.showTasks(req, res).catch(next));
   app.post("/addtask", (req, res, next) => taskList.addTask(req, res).catch(next));
   app.post("/completetask", (req, res, next) => taskList.completeTask(req, res).catch(next));
   app.set("view engine", "jade");

   // catch 404 and forward to error handler
   app.use(function(req, res, next) {
     const err = new Error("Not Found");
     err.status = 404;
     next(err);
   });

   // error handler
   app.use(function(err, req, res, next) {
     // set locals, only providing error in development
     res.locals.message = err.message;
     res.locals.error = req.app.get("env") === "development" ? err : {};

     // render the error page
     res.status(err.status || 500);
     res.render("error");
   });

   module.exports = app;
   ```

3. Finally, save and close the **app.js** file.

## <a name="_Toc395783181"></a>Step 5: Build a user interface

Now let’s build the user interface so that a user can interact with the application. The Express application we created in the previous sections uses **Jade** as the view engine. For more information on Jade, see [Jade language](http://jade-lang.com/).

1. The **layout.jade** file in the **views** directory is used as a global template for other **.jade** files. In this step you will modify it to use [Twitter Bootstrap](https://github.com/twbs/bootstrap), which is a toolkit used to design a website.  

2. Open the **layout.jade** file found in the **views** folder and replace the contents with the following code:

   ```html
   doctype html
   html
     head
       title= title
       link(rel='stylesheet', href='//ajax.aspnetcdn.com/ajax/bootstrap/3.3.2/css/bootstrap.min.css')
       link(rel='stylesheet', href='/stylesheets/style.css')
     body
       nav.navbar.navbar-inverse.navbar-fixed-top
         div.navbar-header
           a.navbar-brand(href='#') My Tasks
       block content
       script(src='//ajax.aspnetcdn.com/ajax/jQuery/jquery-1.11.2.min.js')
       script(src='//ajax.aspnetcdn.com/ajax/bootstrap/3.3.2/bootstrap.min.js')
   ```

    This code tells the **Jade** engine to render some HTML for our application and creates a **block** called **content** where we can supply the layout for our content pages. Save and close the **layout.jade** file.

3. Now open the **index.jade** file, the view that will be used by our application, and replace the content of the file with the following code:

   ```html
   extends layout
   block content
        h1 #{title}
        br
        
        form(action="/completetask", method="post")
         table.table.table-striped.table-bordered
            tr
              td Name
              td Category
              td Date
              td Complete
            if (typeof tasks === "undefined")
              tr
                td
            else
              each task in tasks
                tr
                  td #{task.name}
                  td #{task.category}
                  - var date  = new Date(task.date);
                  - var day   = date.getDate();
                  - var month = date.getMonth() + 1;
                  - var year  = date.getFullYear();
                  td #{month + "/" + day + "/" + year}
                  td
                   if(task.completed) 
                    input(type="checkbox", name="#{task.id}", value="#{!task.completed}", checked=task.completed)
                   else
                    input(type="checkbox", name="#{task.id}", value="#{!task.completed}", checked=task.completed)
          button.btn.btn-primary(type="submit") Update tasks
        hr
        form.well(action="/addtask", method="post")
          label Item Name:
          input(name="name", type="textbox")
          label Item Category:
          input(name="category", type="textbox")
          br
          button.btn(type="submit") Add item
   ```

This code extends layout, and provides content for the **content** placeholder we saw in the **layout.jade** file earlier. In this layout, we created two HTML forms.

The first form contains a table for your data and a button that allows you to update items by posting to **/completeTask** method of the controller.
    
The second form contains two input fields and a button that allows you to create a new item by posting to **/addtask** method of the controller. That's all we need for the application to work.

## <a name="_Toc395783181"></a>Step 6: Run your application locally

1. To test the application on your local machine, run `npm start` in the terminal to start your application, then refresh the [http://localhost:3000](http://localhost:3000) browser page. The page should now look as shown in the following screenshot:
   
    ![Screenshot of the MyTodo List application in a browser window](./media/sql-api-nodejs-application/cosmos-db-node-js-localhost.png)

    > [!TIP]
    > If you receive an error about the indent in the layout.jade file or the index.jade file, ensure that the first two lines in both files is left justified, with no spaces. If there are spaces before the first two lines, remove them, save both files, then refresh your browser window. 

2. Use the Item, Item Name, and Category fields to enter a new task and then select **Add Item**. It creates a document in Azure Cosmos DB with those properties. 

3. The page should update to display the newly created item in the ToDo
   list.
   
    ![Screenshot of the application with a new item in the ToDo list](./media/sql-api-nodejs-application/cosmos-db-node-js-added-task.png)

4. To complete a task, check the checkbox in the Complete column,
   and then select **Update tasks**. It updates the document you already created and removes it from the view.

5. To stop the application, press CTRL+C in the terminal window and then select **Y** to terminate the batch job.

## <a name="_Toc395783182"></a>Step 7: Deploy your application to Azure Websites

1. If you haven't already done, enable a git repository for your Azure Website. You can find instructions on how to enable a git repository in the [Local Git Deployment to Azure App Service](../app-service/app-service-deploy-local-git.md) topic.

2. Add your Azure Website as a git remote.
   
   ```bash
   git remote add azure https://username@your-azure-website.scm.azurewebsites.net:443/your-azure-website.git
   ```

3. Deploy the application by pushing it to the remote.
   
   ```bash
   git push azure master
   ```

4. In a few seconds, your web application is published and launched in a browser.

If you want to download or refer to the complete reference application for this tutorial, you can download it from [GitHub][GitHub].

## <a name="_Toc395637775"></a>Next steps

In this tutorial, you've learned how to build a Node.js web app using JavaScript SDK to manage Azure Cosmos DB SQL API data. You can now proceed to the next article:

> [!div class="nextstepaction"]
> [Build mobile applications with Xamarin and Azure Cosmos DB](mobile-apps-with-xamarin.md)


[Node.js]: https://nodejs.org/
[Git]: https://git-scm.com/
[GitHub]: https://github.com/Azure-Samples/azure-cosmos-db-sql-api-nodejs-todo-app

