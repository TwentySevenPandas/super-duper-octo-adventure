# Creating a survey task from UI Builder user input

This post will walk through how present a user a drop down list of records then create a task based on the selection

I'm going to be doing a series of articles walking through how to build an app that allows a user to schedule a survey to be sent to a group of customers, send them reminders and easily view sentiment and responses.

The idea will be to start slow with a single input and create and slowly build up to something more flashy but complicated

## Assumptions

Thre is an assumption that readers of this post know how to create an expierence, a page, and add a component.  

I'm not going to be starting with a UI Builder `Hello World` because there are heaps great of articles that already go into that at length.. here are some I used to get familar with UI Builder

- [UI Builder Essentials: Dynamically Binding the List Component With a Data Visualization](https://www.servicenow.com/community/next-experience-blog/ui-builder-essentials-dynamically-binding-the-list-component/ba-p/3164521)

- [A UI Builder Experience](https://www.servicenow.com/docs/bundle/xanadu-application-development/page/administer/ui-builder/task/learn-by-example-create-experience.html)

- [In depth guide of how to add components](https://www.servicenow.com/docs/bundle/xanadu-application-development/page/administer/ui-builder/task/add-components.html)


## High level steps

* Create a list of `Asessment Metric Types (asmt_type_type)` records for the user to pick from 
* Add a `Select` component to a page to allow user to see availble surveys and select a survey
* Bind the list of surveys to the component 
* Store the users survey selection
* Create a task with the sys_id of user selected survey recorded in a custom column

## Step 1: Create a List of Assessment Metric Types (asmt_type_type) for User Selection

The ultimate goal here is to allow the user to send a survey to a customer. ServiceNow comes with several out-of-the-box surveys, so for simplicity, we’ll fetch all available surveys from the database.

UI Builder offers data resources (sometimes referred to as "data brokers" in the docs) to manage creating, reading, updating, and deleting records from ServiceNow tables. These resources are powerful and allow us to interact with server-side APIs such as `GlideRecord`, `GlideDateTime`, and `GlideAggregate`.


For this example, we will use the out-of-the-box “Look-up Multiple Records” data resource, which allows us to query the records we need.

### Adding the Data Resource

1. In UI Builder, navigate to the bottom left of the screen under `Data and scripts` and click **Add data resource**.
![](https://live.staticflickr.com/65535/54298418339_42e79c3399_n.jpg)

2. Choose **Look-up multiple records**.  
![](https://live.staticflickr.com/65535/54298612260_5a5a1f652d_n.jpg)

### Configuring the Data Resource

Here’s the configuration I used for this example:

- **Name** and **Label**: Set to `look_up_multiple_assessment_records` for clarity, especially as you add more resources.
- **When to evaluate this data resource**: Choose "Immediately (eager)" to load the available surveys as soon as the page loads.
- **Table**: Set to `asmt_metric_type` to get a list of surveys.
- **Conditions**: `active = true` to ensure only active surveys are included.
- **Return fields**: Return `sys_id` and `display` (display name of the survey).
- **Order by**: Sort by `Name` to keep the records alphabetically ordered.

Once configured, the "Pill view" in UI Builder will display the records fetched by the data resource.

<img src="https://live.staticflickr.com/65535/54300146055_57cab8b8a6.jpg" width="500" height="404" alt="configure a data component"/>


## Step 2: Add a Select Component for User Survey Selection

Next, we’ll add a **Select** component to the page to display the list of available surveys.

1. Click **Add Content** in the Component Tree to the left of the screen.
2. Select **Select** from the list of available components.

The Select component behaves similarly to a reference field in ServiceNow’s backend. It displays a list of labels (which we will configure to be survey names) and values (which we will configure to be a survey `sys_id`) that users can choose from.  
![](https://live.staticflickr.com/65535/54297258772_a29eb75450.jpg)

## Step 3: Bind the Data Resource to the Select Component

We need to bind the data fetched from the data resource to the Select component.

1. Click the newly created Select component in the Component Tree.
2. In the **Configure** tab on the left, scroll down to the **List items** section.
3. Click the **Bind data or use scripts** icon to start the binding process.  
![](https://live.staticflickr.com/65535/54298626090_f1dc7b264f_n.jpg)  
4. In the **Bind data to List items** pop-out, select the **Use script** icon in the top-right corner.  
![](https://live.staticflickr.com/65535/54298296391_b1cc62cab4_n.jpg)

5. Enter the following script to map the survey names to the Select component:  
```
    /**
    * @param {params} params
    * @param {api} params.api
    * @param {TransformApiHelpers} params.helpers
    */
    function evaluateProperty({
        api,
        helpers
    }) {
    // look_up_multiple_assessment_records is the name of the Data resource
    // if you have changed the name of resource change the below code to
    // api.data.** new name of data resource **.results.map((el) => {
            return api.data.look_up_multiple_assessment_records.results.map((el) => {
                return {
                    "id": el._row_data.uniqueValue,
                    "label": el.name.displayValue
                };
            });
    }
```

Once applied, you can preview the page. The Select component should now display the list of surveys fetched from the database.

## Step 4: Storing the User's Survey Selection

To store the survey that the user selects, we’ll need to use a client state parameter to capture the selected `sys_id`.

### Create a Client State Parameter

1. In the **Data and scripts** section, click **Add client state parameter**.
2. Create a new parameter of type **String** and name it `survey_id`.
<img src="https://live.staticflickr.com/65535/54298772895_09f09f6ec7.jpg" width="500" height="66" alt="survey_client_state_id"/>

### Link the Select Component to the Client State Parameter

1. Click the Select component in the Component Tree.
2. In the **Events** tab of the configuration panel to the right of the screen, click **Add event mapping**.
3. Set the **Event** to **Item selected**, and click **Continue**.
4. In the **Handler** section, select **Update client state parameter**.
5. In the **Configure** section, set the **Client State Parameter Name** to `survey_id`.

Now, when the user selects a survey, its `sys_id` will be saved in the `survey_id` client state parameter.




## Step 5: Creating a Task on Button Click

To create a task based on the selected survey, we’ll need to add a **Create Record** data resource.

### Create and configure Survey Task table

For this tutorial and for the rest of this series I will be using custom table that extends `task` in my ServiceNow app called **Survey Tasks** [For information on how to do this check this short youtube video](https://www.youtube.com/watch?app=desktop&v=oRDqN442vW0&t=575s)

On the custom table I will be adding one column called `survey` which will be used to store the `sys_id` of the survey to be sent


### Add a data resource for creating a record

Similar to how the `look up multiple records' resource was created

1. In UI Builder, navigate to the bottom left of the screen under `Data and scripts` and click **Add data resource**.
2. Select `create record`

### Add a Button Component

1. In the Component Tree, click the three dots next to your Select component, and choose **Add After**.
![](https://live.staticflickr.com/65535/54298748105_a741c50b0e_n.jpg)
2. Select **Button** from the list of components.
3. In the configure" tab of the configuration panel to the right hand side of the screen change the label from Button to Submit

### Configure the Button to Create a Task

1. In the configurtion panel to the right of the screen click the **Events** tab
2. Click **Add handler** under  **Button clicked** 
3. Scroll down and select the `create record ` resource you just created, then click continue
<img src="https://live.staticflickr.com/65535/54299766181_3c7a2dd33a.jpg" width="500" height="283"/>  
4. In the `Table` field start typing **Survey Task* 
5. Click **Edit field values**

> NOTE: the Create and Update Data resoruces use a condition builder to set fields 

6. Select `survey` column we created on our **Survey Tasks** table in the left most drop down
7. Select `is` in the middle column
8. Using the bind data icon by hovering over the right most colmn click the `bind data` icon
<img src="https://live.staticflickr.com/65535/54300218420_b2010768cb.jpg" width="500" height="115" />

9. Select the `survey_id` **Client state parameter** you created earlier, click the little arrow next to it (the pill should be copied into the box above), then click apply  
<img src="https://live.staticflickr.com/65535/54298919692_e52f00ab49.jpg" width="500" height="188" />

To test goto preview in the top right hand side of UI builder, pick a survey and click submit. If you check the table you created you should see a new record 

## Step 6: Displaying results to user

We are now going to repeat some of the steps above with minor changes to display information to the user about the newly created task. 

1. Create a new **Client state parameter** called `task_number` from **Data and scripts** from the bottom right hand corner of UI Builder interface
2. Create a **Client Script** called `store_task_number` from the **Data and scripts** from the bottom right hand corner of UI Builder interface and paste the below code into the script box

``` 
/**
* @param {params} params
* @param {api} params.api
* @param {any} params.event
* @param {any} params.imports
* @param {ApiHelpers} params.helpers
*/
function handler({api, event, helpers, imports}) {
    /**
    In the below example my "Survey Task" table is called x_667488_surveys_survey_task
    For this to work in your instance you will need to change insert_x_667488_surveys_survey_task to what ever your instance is called
    **/

    const id = event.payload.data.output.data.GlideRecord_Mutation.insert_x_667488_surveys_survey_task.sys_id.value;
    api.setState('task_id', id);
}

```

3. Click the *Create Record* data resource
4. Under the *Events* tab on the pop-out click *Add event mapping*
5. Choose *Operation Succeeded* then click continue in the bottom right hand corner of the pop-out
6. Scroll down to find `store_task_number` **Client Script** under the *Client Scripts** section
<img src="https://live.staticflickr.com/65535/54300049714_74a09527d7.jpg" width="393" height="487" alt="add client script"/>
7. add a new **Rich text component** from the **Component tree**
8. Bind the `task_number` **Client state parameter** to the **Rich Text Component**
<img src="https://live.staticflickr.com/65535/54299824451_870b02b884_n.jpg" width="319" height="235" alt="bind to html"/>

Save, click preview, and the `sys_id` of the newly created task will be shown on screen

If you wanted more information from the task a **Look up single record** **Data Resource** could be triggered from the ** Operation succeeded * event

# Next Time

In the next tutorial  will focus on date time selectors. We will restrict what dates users can select, and create a custom data resource to work out when reminders and survey expiry should be 

To [download an date set of this tutorial plus some extra styling goto ](https://github.com/TwentySevenPandas/super-duper-octo-adventure/tree/main/1%20-%20Survey%20App%20Creating%20a%20survey%20task)
