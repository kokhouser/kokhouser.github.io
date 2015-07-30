---
layout: post
title: How To Use Some Essential Android Libraries - Part I
categories: []
tags: []
published: True

---

<p>
	With some free time on my hands, and some plans to spin up Android development again, I've gone ahead and experimented with some API's and libraries I've been reading about a lot. One thing led to another and I now have an app on the Play Store, mainly as an example on the libraries detailed in this post. For shameless reference however, you can find my app <a href="">here</a>. <br> <br>
	The app is also fully open-source code, with the source code available <a href="https://github.com/kokhouser/NAPOD">here</a>. <br> <br>
	So without further ado, I'll write a bit about how to go about recreating this app, which is a image viewer for <a href="https://api.nasa.gov/api.html#apod">Nasa's Astronomy Picture of The Day</a>. This app also can be used to download the pictures, as well as integrate fully with <a href="https://play.google.com/store/apps/details?id=net.nurik.roman.muzei&hl=en">Muzei</a>, a popular live wallpaper by the talented <a href="http://roman.nurik.net/">Roman Nurik</a>. With Muzei integration, it's possible to set your wallpaper to pull a new NAPOD image daily, as well as set up a custom watchface for your Android Wear device if you happen to have one.<br><br>
	Note: This post assumes we're using Android Studio 1.3 with the latest Android SDK. The entire process detailing how I created the app and published it to the Play Store would be detailed in several parts, with this one talking about obtaining data from REST API's and creating Java objects with the data using Retrofit and GSON
</p>
	<img src="/assets/images/NAPOD/screenshot.png" style="max-width:95%;height:auto;">
<p>
	The libraries we'll be exploring in this series of posts are:
	<ul>
		<li><a href="http://square.github.io/retrofit/">Retrofit</a>: Square's amazing REST API library. This turns REST API's into Java interfaces we can work with. It also handles everything off the main thread so we don't have to worry about pesky AsyncTasks.</li>
		<li><a href="http://square.github.io/okhttp/">OKHTTP</a>: A dependency of Retrofit. Aids HTTP requests.</li>
		<li><a href="https://github.com/google/gson">GSON</a>: A dependency of Retrofit. Serializes JSON objects into simple Java objects we'll create.</li>
		<li><a href="http://square.github.io/picasso/">Picasso</a>: Another amazing library from Square. We'll use it to handle image downloading, and again it does everything off the main thread.</li>
		<li><a href="https://github.com/code-mc/material-icon-lib">Material Icon Library</a>: A fancy library that adds a new view inheriting from the standard ImageView, and lets us include many icons designed with Google's Material Design spec.</li>
		<li><a href="https://github.com/JakeWharton/butterknife">Butterknife</a>: An extremely handy view injection library that cuts down on all the verbose view finding we'll do in the activities.</li>
		<li><a href="https://github.com/romannurik/muzei/wiki/API">Muzei Live Wallpaper API</a>: The API that lets us specify our app as a Muzei wallpaper source.</li>
	</ul>
</p>
	<br>
<p>
	First, we'll include all the required libraries in our app.gradle file. I should also mention that we'll need a NASA API key, which we can get <a href="https://api.nasa.gov/index.html#getting-started">here</a> or use the default demo key.
	<pre><code>
		dependencies {
		    compile fileTree(dir: 'libs', include: ['*.jar'])
		    compile 'com.android.support:appcompat-v7:22.2.0'
		    compile 'com.squareup.okhttp:okhttp:2.4.0'
		    compile 'com.squareup.retrofit:retrofit:1.9.0'
		    compile 'com.google.code.gson:gson:2.3'
		    compile 'com.squareup.picasso:picasso:2.5.2'
		    compile 'net.steamcrafted:materialiconlib:1.0.3'
		    compile 'com.jakewharton:butterknife:6.1.0'
		    compile 'com.android.support:design:22.2.0'
		    compile 'com.google.android.apps.muzei:muzei-api:+'
		}
	</code></pre>
	<br>
	Since we'll only be talking about Retrofit and GSON today, we can get by the rest of this post with only the following dependencies:-
	</p>
	<br>
	<pre><code>
		dependencies {
		    compile fileTree(dir: 'libs', include: ['*.jar'])
		    compile 'com.squareup.okhttp:okhttp:2.4.0'
		    compile 'com.squareup.retrofit:retrofit:1.9.0'
		    compile 'com.google.code.gson:gson:2.3'
		}
	</code></pre>
	<br>
<p>
	Next, we'll start by creating the model we need to store out astronomy picture, in this case I called it an Astropic. It would help if we created a "Models" directory under our project in Android Studio.

	<br>
	<br>
	<img src="/assets/images/NAPOD/project_structure.png" style="max-width:95%;height:auto;">
	<br>
	<br>

	Retrofit and GSON makes out lives easier by handling all the JSON to Java Object conversion, however we must first define the Java object class for the Astropic. To do that, we must also first know what the response from the API looks like. Fortunately, this is very easy to do, just navigate to <a href = "https://api.nasa.gov/planetary/apod?api_key=DEMO_KEY">https://api.nasa.gov/planetary/apod?api_key=DEMO_KEY</a> and you should see something that looks like this: -

	<pre><code>
{"url": "http://apod.nasa.gov/apod/image/1507/MWUluruTafreshi_MG_6685Ps.jpg", "media_type": "image", "explanation": "The central regions of our Milky Way Galaxy rise above Uluru/Ayers Rock in this striking night skyscape. Recorded on July 13, a faint airglow along the horizon shows off central Australia's most recognizable landform in silhouette. Of course the Milky Way's own cosmic dust clouds appear in silhouette too, dark rifts along the galaxy's faint congeries of stars. Above the central bulge, rivers of cosmic dust converge on a bright yellowish supergiant star Antares. Left of Antares, wandering Saturn shines in the night.", "concepts": null, "title": "Milky Way over Uluru"}
	</code></pre>
	<br>
	This will be our sample JSON which we'll use to generate code that describes our Java object, through a wonderful tool called <a href ="http://www.jsonschema2pojo.org/">JSONSchema2POJO</a>. Paste the JSON output into the box there, select "JSON" instead of "JSON Schema" under "Source Type", and "None" under "Annotation style". Accept the rest of the defaults and click "Preview". Copy what you see into a Java class in Android Studio, under Models.
	</p>
	<p>
	<pre><code>
	package com.kokhouser.napod.models;

	import com.google.gson.annotations.SerializedName;

	import java.util.HashMap;
	import java.util.Map;

	/**
	 * Created by hkok on 7/28/2015.
	 * Using code generated from http://www.jsonschema2pojo.org/
	 */

	public class Astropic {
	    private String url;
	    @SerializedName("media_type")
	    private String mediaType;
	    private String explanation;
	    private Object concepts;
	    private String title;
	    private Map<String, Object> additionalProperties = new HashMap<String, Object>();

	    /**
	     *
	     * @return
	     * The url
	     */
	    public String getUrl() {
	        return url;
	    }

	    /**
	     *
	     * @param url
	     * The url
	     */
	    public void setUrl(String url) {
	        this.url = url;
	    }

	    /**
	     *
	     * @return
	     * The mediaType
	     */
	    public String getMediaType() {
	        return mediaType;
	    }

	    /**
	     *
	     * @param mediaType
	     * The media_type
	     */
	    public void setMediaType(String mediaType) {
	        this.mediaType = mediaType;
	    }

	    /**
	     *
	     * @return
	     * The explanation
	     */
	    public String getExplanation() {
	        return explanation;
	    }

	    /**
	     *
	     * @param explanation
	     * The explanation
	     */
	    public void setExplanation(String explanation) {
	        this.explanation = explanation;
	    }

	    /**
	     *
	     * @return
	     * The concepts
	     */
	    public Object getConcepts() {
	        return concepts;
	    }

	    /**
	     *
	     * @param concepts
	     * The concepts
	     */
	    public void setConcepts(Object concepts) {
	        this.concepts = concepts;
	    }

	    /**
	     *
	     * @return
	     * The title
	     */
	    public String getTitle() {
	        return title;
	    }

	    /**
	     *
	     * @param title
	     * The title
	     */
	    public void setTitle(String title) {
	        this.title = title;
	    }

	    public Map<String, Object> getAdditionalProperties() {
	        return this.additionalProperties;
	    }

	    public void setAdditionalProperty(String name, Object value) {
	        this.additionalProperties.put(name, value);
	    }

	}
	</code></pre>
</p>
<p>
	<br>
	Now that we have our Java object defined, we set up Retrofit and GSON to handle getting data in them.<br>
	<br>
	A quick look at the APOD API shows us that we'll need to call it with a few parameters. The ones we'll care about are "API_KEY" and "DATE". To create a Java interface for this API, we'll create it like so:- 
</p>
<p>
	<code><pre>
		package com.kokhouser.napod.api;

		import com.kokhouser.napod.models.Astropic;

		import retrofit.Callback;
		import retrofit.http.GET;
		import retrofit.http.Query;

		/**
		 * Created by hkok on 7/28/2015.
		 * Retrofit API Interface for the NASA APOD API
		 */
		public interface APODAPIInterface {
		    @GET("/planetary/apod")
		    void getPictureWithKey(@Query("api_key")String apiKey, Callback<Astropic> cb);

		    @GET("/planetary/apod")
		    void getPictureWithKeyAndDate(@Query("api_key")String apiKey, @Query("date")String date, Callback<Astropic> cb);
		}
	</code></pre>
</p>
<p>
	Note that both of our API endpoints will be called asynchonously and thus have a callback function it'll call once it's done. <br>
	<br>
	Finally, we'll need to call this little API interface we've created. Creatively, we'll create an APICaller class to get this done:-
</p>
<p>
	<code><pre>
	package com.kokhouser.napod.api;

	/**
	 * Created by hkok on 7/28/2015.
	 * Class to handle API requests
	 */

	public class APICaller {

	    private static final String BASE_URL = "https://api.nasa.gov";
	    private RestAdapter restAdapter;
	    private APODAPIInterface apodapiInterface;

	    public APICaller(){
	        restAdapter = new RestAdapter.Builder()
	                .setLogLevel(RestAdapter.LogLevel.FULL)
	                .setEndpoint(BASE_URL)
	                .build();
	        apodapiInterface = restAdapter.create(APODAPIInterface.class);
	    }

	    public void callPictureAPIWithKey(final String key){
	        //Prepare view for API call here;
	        apodapiInterface.getPictureWithKey(key, new Callback<Astropic>() {
	            @Override
	            public void success(Astropic astropic, Response response) {
	                if (astropic.getMediaType() != null && astropic.getMediaType().equals("video")) {
	                    //Get another pic
	                } else {
	                    //Do something with astropic here
	                }
	            }

	            @Override
	            public void failure(RetrofitError error) {
	                error.printStackTrace();
	            }
	        });
	    }

	    public void callPictureAPIWithKeyAndDate(final String key, String date){
	        apodapiInterface.getPictureWithKeyAndDate(key, date, new Callback<Astropic>() {
	            @Override
	            public void success(Astropic astropic, Response response) {
	                if (astropic.getMediaType() != null && astropic.getMediaType().equals("video")) {
	                    //Get another pic
	                } else {
	                    //Do something with astropic here
	                }
	            }

	            @Override
	            public void failure(RetrofitError error) {
	                error.printStackTrace();
	            }
	        });
	    }
	}
	</code></pre>
</p>
<p>
	Wow, that's a lot to process. Note that the commented lines indicate future code that we'll add to handle various results. Suffice to say, at this point we'll have set up the APICaller such that the variable "astropic" contains the Astropic object we'll need. <br>
	<br>
	In the next part (whenever I get around to writing that up), we'll look at setting up our activity that uses this API caller. We'll also look into using Picasso to download the pictures provided to us within the Astropic.
</p>