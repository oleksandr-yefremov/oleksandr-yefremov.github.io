---
layout: post
title: "Scheduling repetitive task on Android"
date: 2014-11-01 11:40:53 +0300
comments: false
categories: android
published: true
---
<script src="{{ root_url }}/javascripts/expand-collapse.js" type="text/javascript"></script>

There is a task that needs to be repeated every *N* minutes. Might be even an interview question to see what options person would consider. With no other requirements or restrictions given, here is what pops up in my mind:

<!-- more -->
As a task example I 

**1. Service with Handler**  

<a class="expander" href="#">Source code</a>
<div class="expandable-content">
```java	
public class CurrencyManagerService extends Service {

	private static Logger log = LoggerFactory.getLogger(CurrencyManagerService.class);

	private static final long TIMEOUT_FETCH_CURRENCY_RATES = 1000 * 5; // seconds
	private static long ERROR_BACKOFF_TIMEOUT = TIMEOUT_FETCH_CURRENCY_RATES; // seconds
	private static final String PREFS_KEY_CURRENCY_RATES = ".currencyRates";

	private static Handler handler;

	@Override
	public void onCreate() {
		super.onCreate();
		HandlerThread fetchThread = new HandlerThread("fetchThread");
		fetchThread.setPriority(Thread.MIN_PRIORITY);
		fetchThread.start();

		handler = new Handler(fetchThread.getLooper());
	}

	@Override
	public int onStartCommand(Intent intent, int flags, int startId) {
		log.debug("Service started.");

		// firstly, check if there is sticky event available already
		// (service may be restarted by system occasionally)
		CurrencyRateResponse currencyRateResponse = EventBus.getDefault().getStickyEvent(CurrencyRateResponse.class);
		if (currencyRateResponse == null) {
			// then check if we have it stored in SharedPreferences
			currencyRateResponse = PreferencesToGson.getInstance(getApplicationContext())
			                                        .getObject(PREFS_KEY_CURRENCY_RATES, CurrencyRateResponse.class);
			if (currencyRateResponse == null) {
				// app fresh start, go fetch rates now
				handler.post(fetchCurrencyRatesRunnable);
			} else {
				// found in SharedPreferences; post event and schedule fetch meanwhile to update cache
				EventBus.getDefault().postSticky(currencyRateResponse);
				handler.post(fetchCurrencyRatesRunnable);
			}
		} else {
			// sticky event is there, schedule next fetch
			handler.postDelayed(fetchCurrencyRatesRunnable, TIMEOUT_FETCH_CURRENCY_RATES);
		}

		return START_STICKY;
	}

	private Runnable fetchCurrencyRatesRunnable = new Runnable() {
		@Override
		public void run() {
			log.debug("Fetching CurrencyRate…");
			RestAdapter restAdapter = new RestAdapter.Builder()
					                          .setEndpoint("http://btchouse.com:8080")
					                          .build();
			CurrencyRatesRestService restService = restAdapter.create(CurrencyRatesRestService.class);
			restService.getRate("USD", currencyRatesRequestCallback);
		}
	};

	private Callback<CurrencyRateResponse.CurrencyRate> currencyRatesRequestCallback = new Callback<CurrencyRateResponse.CurrencyRate>() {
		@Override
		public void success(CurrencyRateResponse.CurrencyRate currencyRate, Response response) {
			ERROR_BACKOFF_TIMEOUT = TIMEOUT_FETCH_CURRENCY_RATES; // set error timeout to default value
			log.info(currencyRate.toString());

			CurrencyRateResponse currencyRateResponse = new CurrencyRateResponse(System.currentTimeMillis(), currencyRate);
			PreferencesToGson.getInstance(getApplicationContext())
			                 .putObject(PREFS_KEY_CURRENCY_RATES, currencyRateResponse);
			EventBus.getDefault().postSticky(currencyRateResponse);
			handler.postDelayed(fetchCurrencyRatesRunnable, TIMEOUT_FETCH_CURRENCY_RATES);
		}

		@Override
		public void failure(RetrofitError error) {
			log.error("Error while fetching currency rates.", error);
			// schedule next fetch with some backoff delay
			handler.postDelayed(fetchCurrencyRatesRunnable, ERROR_BACKOFF_TIMEOUT *= 2);
		}
	};

	@Nullable
	@Override
	public IBinder onBind(Intent intent) {
		// we don't allow binding to this service, since it doesn't make much sense
		// (i.e. other components don't need to send messages to service)
		return null;
	}

	}

```
</div>

	
**2. TimerTask**  
Android doc for [TimerTask](http://developer.android.com/reference/java/util/Timer.html) says "Prefer [`ScheduledThreadPoolExecutor`](http://developer.android.com/reference/java/util/concurrent/ScheduledThreadPoolExecutor.html) for new code." 
	
<a href="/images/throttling/IceFloor Interfaces.png"><img class="caption" style="background-color:white;" src="/images/throttling/IceFloor Interfaces.png" width="374" height="214 " /></a>

**3. ScheduledThreadPoolExecutor**

**4. AlarmManager**
From the [AlarmManager](http://developer.android.com/reference/android/app/AlarmManager.html) doc:  
Note: The Alarm Manager is intended for cases where you want to have your application code run at a specific time, even if your application is not currently running. For normal timing operations (ticks, timeouts, etc) it is easier and much more efficient to use Handler.

