# Couchbase by Example: Android Recycler View Animations

If there’s one big take away from the Android L release is that motion matters. Movement can teach a user what something can do and where it came from. By using motion we can teach users how the system behaves and what they can expect from that system.

## Recycler View

Recycler View is a base for new adapter backed views. It’s hard to get any sort of rich experience with the ListView API.

RecyclerView is meant to have more flexible APIs for the large datasets that you would traditionally use a ListView for. For example, you can now notify the adapter when items are specifically added, removed rather than saying "hey. my dataset changed". That way we can benefit from animations when adding, removing items to the set.

## Getting Started

In this tutorial, you will use a RecyclerView to display a list of restaurants in London. You will use the Google Places API to import documents to Sync Gateway. The information you will be displaying on the screen are the restaurant name, address and thumbnail.

Open `Android Studio` and create a new project called `CityExplorer`:

![][image-1]

Select the `Phone and Tablet` form factor and `API 14` for the minimum SDK.

![][image-2]

Select the `Blank Activity` template:

![][image-3]

Run the app and you should see the default activity and toolbar:

![][image-4]

## How to write a material app?

You will use the support library. It already included things like the drawer layout, view pager. In the L release, the Android UI Toolkit team added support for the RecyclerView and CardView widgets. That way, you can use the new APIs and your application and have a nice fallback on older platform versions.

For apps on previous versions, use the AppCompat library which has been expanded to cover the material design components in L.

## Setting up Sync Gateway

First you need to have a Sync Gateway instance running with documents (including attachments) to replicate. You will use the NodeJS app from `04-ios-sync-progress-indicator` to do that.

Download Sync Gateway and unzip the file:

> http://www.couchbase.com/nosql-databases/downloads#Couchbase\_Mobile

Start Sync Gateway with the config file from `04-ios-sync-progress-indicator`:

	$ ~/Downloads/couchbase-sync-gateway/bin/sync_gateway ~/couchbase-by-example/04-ios-sync-progress-indicator/04-ios-sync-progress-indicator.json

Open the Admin Dashboard to monitor the documents that were saved to Sync Gateway.

	http://localhost:4985/_admin/

In the 04 folder in Terminal, first install the babel node module:

	npm install babel-node -g

And run the script to import the restaurant data from the Google Places API to Sync Gateway:

	babel-node sync.js

Back in the Admin Dashboard, you should now see a bunch of documents.

## Android Design Support Library

You will use the Design Support Library available in the Android M developer preview.

Open the Android SDK manager and install the Android Support Library package.

In the `build.gradle`, add the reference to the design library.

	compile 'com.android.support:design:22.2.0'

##  Floating Action Button

You will use the Floating Action Button available in the Design Support Library to remove items in the RecyclerView.

Copy the [add and delete icons][1] in your project.

In `main_layout.xml`, add the FAB button XML:

	<android.support.design.widget.FloatingActionButton
	    android:layout_width="wrap_content"
	    android:layout_height="wrap_content"
	    android:layout_alignParentBottom="true"
	    android:layout_alignParentRight="true"
	    android:layout_marginBottom="16dp"
	    android:layout_marginRight="16dp"
	    android:src="@drawable/ic_add_white_24dp" />
	
	<android.support.design.widget.FloatingActionButton
	    android:layout_width="wrap_content"
	    android:layout_height="wrap_content"
	    android:layout_alignParentBottom="true"
	    android:layout_alignParentLeft="true"
	    android:layout_marginBottom="16dp"
	    android:layout_marginLeft="16dp"
	    android:onClick="deletePlace"
	    android:src="@drawable/ic_delete_white_24dp" />

Run the app and you should see both button:

![][image-5]

In the next section, you will set the Android app to pull those documents and display them in the RecyclerView.

## Replication

In the `MainActivity`, add a method to register a views to index documents:

	private void registerViews() {
	    View placesView = database.getView(PLACES_VIEW);
	    placesView.setMap(new Mapper() {
	        @Override
	        public void map(Map<String, Object> document, Emitter emitter) {
	            emitter.emit(document.get("_id"), document);
	        }
	    }, "1");
	}

In the `onCreate` method, add some code setup the replication:

	try {
	    // replace with the IP to use
	    URL url = new URL("http://192.168.1.218:4984/db");
	    Manager manager = new Manager(new AndroidContext(getApplicationContext()), Manager.DEFAULT_OPTIONS);
	    database = manager.getExistingDatabase("cityexplorer");
	    if (database != null) {
	        database.delete();
	    }
	    database = manager.getDatabase("cityexplorer");
	    registerViews();
	    Replication pull = database.createPullReplication(url);
	    pull.setContinuous(true);
	    pull.start();
	} catch (MalformedURLException e) {
	    e.printStackTrace();
	} catch (CouchbaseLiteException e) {
	    e.printStackTrace();
	} catch (IOException e) {
	    e.printStackTrace();
	}

**NOTE**: Don’t forget to replace the hostname accordingly.

Run the application and you should the list of changes occurring during the replication in LogCat:

![][image-6]

## RecyclerView

Open `activity_main.xml` and add the following inside the `LinearLayout` tag:

	<android.support.v7.widget.RecyclerView
	    android:id="@+id/list"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent" />

In `MainActivity`, add a `recyclerView` property of type `RecyclerView` and initialise in the `onCreate` method:

	recyclerView = (RecyclerView) findViewById(R.id.list);
	recyclerView.setLayoutManager(new LinearLayoutManager(this));

In the next section, you will add the XML file that represents the UI for each row in the RecyclerView.

## RecyclerView Rows

Each row in the RecyclerView will have an `ImageView` and 2 `TextViews`:

	wireframe

In the `res/layout` directory, create a new Layout resource file and call it `row_places.xml` and paste the following XML:

	<?xml version="1.0" encoding="utf-8"?>
	<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:layout_width="match_parent"
	    android:layout_height="wrap_content"
	    android:layout_marginTop="10dp"
	    android:layout_marginBottom="10dp">
	
	    <ImageView
	        android:id="@+id/restaurantImage"
	        android:layout_width="40dp"
	        android:layout_height="40dp"
	        android:src="@drawable/ic_add_white_24dp" />
	
	    <LinearLayout
	        android:layout_marginLeft="10dp"
	        android:layout_width="0dp"
	        android:layout_height="wrap_content"
	        android:layout_weight="1"
	        android:orientation="vertical">
	
	        <TextView
	            android:id="@+id/restaurantName"
	            android:layout_width="match_parent"
	            android:layout_height="wrap_content"
	            android:text="This is the title" />
	
	        <TextView
	            android:id="@+id/restaurantText"
	            android:layout_width="match_parent"
	            android:layout_height="wrap_content"
	            android:text="This is the description" />
	    </LinearLayout>
	
	</LinearLayout>


In the next section, you will implement the adapter class for the Recycler View.

## Implementing the Adapter

Add a Java class called `PlacesAdapter`. Add the constructor and implement the `onCreateViewHolder` and `onBindViewHolder` methods.

You can find the content of the file [here][2].

Notice that the constructor of `PlacesAdapter` takes a few arguments in addition to the context:

- `List<Place>` dataSet: the list of documents to display on screen. Place is model class that you will create in the next section.
- `Database` database: the database object to get the attachment and populate the `ImageView` view.

In the next section, you will create the Place model class.

## Model

The rows returned by a map/reduce query contain a key and value. The value is of type `LazyJsonObject`. We can use this class for parsing the JSON data. Open a new file named `Place` extending `LazyJsonObject` with the following getter methods:

	public class Place {
	    private LazyJsonObject mLazy;
	
	    public Place(LazyJsonObject lazyJsonObject) {
	        mLazy = lazyJsonObject;
	    }
	
	    public String getName() {
	        return (String) mLazy.get("name");
	    }
	
	    public String getId() {
	        return (String) mLazy.get("_id");
	    }
	
	    public String getAddress() {
	        return (String) mLazy.get("formatted_address");
	    }
	}

##  Connecting the Adapter to the RecyclerView

Back in the `onCreate` of `MainActivity`, initialise the adapter property and connect it to the recycler view:

	adapter = new PlacesAdapter(this, new ArrayList<Place>(), database);
	recyclerView.setAdapter(adapter);

Update the database listener inner class with the following code to reload the RecyclerView:

	database.addChangeListener(new Database.ChangeListener() {
	    @Override
	    public void changed(Database.ChangeEvent event) {
	        if (event.isExternal()) {
	            QueryEnumerator rows = null;
	            try {
	                rows = queryPlaces.run();
	            } catch (CouchbaseLiteException e) {
	                e.printStackTrace();
	            }
	            List<Place> places = new ArrayList<>();
	            for (Iterator<QueryRow> it = rows; it.hasNext(); ) {
	                QueryRow row = it.next();
	                Log.d("", row.getValue().toString());
	                Map<String, Object> properties = database.getDocument(row.getDocumentId()).getProperties();
	                places.add(new Place((LazyJsonObject) row.getValue()));
	            }
	            adapter.dataSet = places;
	            runOnUiThread(new Runnable() {
	                @Override
	                public void run() {
	                    recyclerView.getAdapter().notifyDataSetChanged();
	                }
	            });
	        }
	    }
	});

## Deleting items

Implement the `deletePladce` method in MainActivity:

	public void deletePlace(android.view.View view) {
	    Log.d("", "delete me");
	    adapter.dataSet.remove(2);
	    try {
	        database.getExistingDocument(adapter.dataSet.get(2).getId()).delete();
	    } catch (CouchbaseLiteException e) {
	        e.printStackTrace();
	    }
	    adapter.notifyItemRemoved(2);
	}

Run the app and delete items with animations:

![][image-7]

Use the `babel-node sync.js` again to add another set of 20 documents with attachments and notice the RecyclerView will reload without animations:

![][image-8]

## Conclusion

In this tutorial you learned how to use a database change listener to re-run the query backing a Recycler View. In addition, you learned to use the various RecyclerView APIs to include system level support.

It would be interesting to explore further the new APIs available in the L release such Shadows to elevate your views. You can post certain views above the view hierarchy plane. More precisely, with this API, you can boost some of those views with an elevation `z` value that puts them above the plane. Coupling shadows with change notifications during a replication for example could yield a great user experience.

## N.B: When to use a Live Query?

A live query stays active and monitors the database and view index for changes. When there’s a change it re-runs itself automatically, and if the query results changed it notifies any observers.

So you will be notified that things have changed and passed a `ChangeEvent`. The live query `ChangeEvent` object has the following getter methods:

- `getSource`: returns the associated live query
- `getError`: returns the error if any
- `getRows`: returns the result as a `QueryEnumerator` (i.e. `Iterator<QueryRow>`)

You can retrieve the new result with the `getRows` method. This method returns all the rows for that query, not only the ones that changed.

To make the most out of the RecyclerView, we would like to know which item(s) were added/removed or just updated in order to use the item animator and have nice animations like below:

	gif 

A Live Query listens on the Database Change Listener, every time the Database Change Listener is triggered, the query will run again.

However, if you want to build reactive UI with animations, we need to know which documents have changed and the database change event listener gives us this ability.

And this brings us to the next section to discuss the different scenarios when the adapter should tell the RecyclerView to redraw itself.

## Database Change Listener

Have a look at the diagram below:

![][image-9]

The requirements we will follow are:

- When the RecyclerView loads for the first time, all items should be animated
- When the user makes a change, choose the appropriate RecyclerView API call to trigger the animation
- When a document is replicated from Sync Gateway, reload the RecyclerView without animations

A live query is listening on the database change listener. A `ChangeEvent` object on the database listener has very interesting properties like `external` to indicate if the change was the result of a replication or of a local CRUD operation.

In the next section, you will set up the replication to handle different operations.

[1]:	google.com
[2]:	http://google.com

[image-1]:	http://cl.ly/image/1S1G2b0M1k1s/Screen%20Shot%202015-06-26%20at%2011.28.42.png
[image-2]:	http://cl.ly/image/1N0b1t3N1e1P/Screen%20Shot%202015-06-26%20at%2011.44.51.png
[image-3]:	http://cl.ly/image/2R3y2e0e0F2j/Screen%20Shot%202015-06-26%20at%2011.45.34.png
[image-4]:	http://cl.ly/image/2m3E2h2O0l40/shamuLMY47Zjamesnocentini06262015115529.png
[image-5]:	http://cl.ly/image/2R211W2w1K10/shamuLMY47Zjamesnocentini06262015123903.png
[image-6]:	http://cl.ly/image/0N3O3V371g3L/Screen%20Shot%202015-06-26%20at%2012.49.05.png
[image-7]:	http://cl.ly/image/220Y221s2h2y/Untitled.gif
[image-8]:	http://cl.ly/image/162X0w0P1G0I/replication.gif
[image-9]:	http://cl.ly/image/281j3X1m2s1j/Recycler%20View%20(1).png