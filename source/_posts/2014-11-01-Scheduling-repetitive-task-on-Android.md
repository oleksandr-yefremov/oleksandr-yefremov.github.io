---
layout: post
title: "Scheduling repetitive task on Android"
date: 2014-11-01 11:40:53 +0300
comments: false
categories: android
published: true
---
<script src="{{ root_url }}/javascripts/expand-collapse.js" type="text/javascript"></script>

There is a task that needs to be repeated every *N* seconds. Might be even an interview question to see what options person would consider. With no other requirements or restrictions given, here is what pops up in my mind:

<!-- more -->
As a task example I 

**1. Service with Handler**  


<div class="expand-container">
<a class="expander" href="#">Expand source code</a>
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
</div></div>

<div class="expand-container">
<a class="expander" href="#">Memory footprint</a>
<div class="expandable-content">

<a href="/images/scheduling_task/nexus5_device/1.png"><img class="caption" src="/images/scheduling_task/nexus5_device/1.png" /></a>
<a href="/images/scheduling_task/nexusS_genymotion/1e.png"><img class="caption" src="/images/scheduling_task/nexusS_genymotion/1e.png" /></a>

</div></div>
	

**2. TimerTask**  
Android doc for [TimerTask](http://developer.android.com/reference/java/util/Timer.html) says "Prefer [`ScheduledThreadPoolExecutor`](http://developer.android.com/reference/java/util/concurrent/ScheduledThreadPoolExecutor.html) for new code." 

<div class="expandable-container">	
<a class="expander" href="#">Expand source code</a>
<div class="expandable-content">
```java	
public class CurrencyManagerTimer {
	private static Logger log = LoggerFactory.getLogger(CurrencyManagerTimer.class);

	private static final long TIMEOUT_FETCH_CURRENCY_RATES = 1000 * 5; // seconds
	private static long ERROR_BACKOFF_TIMEOUT = TIMEOUT_FETCH_CURRENCY_RATES;
	private static final String PREFS_KEY_CURRENCY_RATES = ".currencyRates";

	private final Context context;

	private static Timer timer;
	private TimerTask timerTask;

	public CurrencyManagerTimer(@NonNull Context context) {
		this.context = context;
		timer = new Timer("fetchTimer");
		timerTask = createTimerTask();
		start();
	}

	private TimerTask createTimerTask() {
		return new TimerTask() {
			@Override
			public void run() {
				log.debug("Fetching CurrencyRate…");
				RestAdapter restAdapter = new RestAdapter.Builder()
						                          .setEndpoint("http://localhost:8080")
						                          .build();
				CurrencyRatesRestService restService = restAdapter.create(CurrencyRatesRestService.class);
				restService.getRate("USD", currencyRatesRequestCallback);
			}
		};
	}

	private Callback<CurrencyRateResponse.CurrencyRate> currencyRatesRequestCallback = new Callback<CurrencyRateResponse.CurrencyRate>() {
		@Override
		public void success(CurrencyRateResponse.CurrencyRate currencyRate, Response response) {
			ERROR_BACKOFF_TIMEOUT = TIMEOUT_FETCH_CURRENCY_RATES; // set error timeout to default
			log.info(currencyRate.toString());

			CurrencyRateResponse currencyRateResponse = new CurrencyRateResponse(System.currentTimeMillis(), currencyRate);
			PreferencesToGson.getInstance(context)
			                 .putObject(PREFS_KEY_CURRENCY_RATES, currencyRateResponse);
			EventBus.getDefault().postSticky(currencyRateResponse);
		}

		@Override
		public void failure(RetrofitError error) {
			log.error("Error while fetching currency rates.", error);
			// schedule next fetch with some backoff delay
			timerTask.cancel();
			timerTask = createTimerTask();
			timer.scheduleAtFixedRate(timerTask, ERROR_BACKOFF_TIMEOUT *= 2, TIMEOUT_FETCH_CURRENCY_RATES);
		}
	};


	private void start() {
		log.debug("Manager started.");

		// firstly, check if there is sticky event available already
		CurrencyRateResponse currencyRateResponse = EventBus.getDefault().getStickyEvent(CurrencyRateResponse.class);
		if (currencyRateResponse == null) {
			// then check if we have it stored in SharedPreferences
			currencyRateResponse = PreferencesToGson.getInstance(context)
			                                        .getObject(PREFS_KEY_CURRENCY_RATES, CurrencyRateResponse.class);
			if (currencyRateResponse == null) {
				// app fresh start, go fetch rates now
				timer.scheduleAtFixedRate(timerTask, 0, TIMEOUT_FETCH_CURRENCY_RATES);
			} else {
				// found in SharedPreferences; post event and schedule fetch meanwhile to update cache
				EventBus.getDefault().postSticky(currencyRateResponse);
				timer.scheduleAtFixedRate(timerTask, 0, TIMEOUT_FETCH_CURRENCY_RATES);
			}
		} else {
			// sticky event is there, schedule next fetch
			timer.scheduleAtFixedRate(timerTask, TIMEOUT_FETCH_CURRENCY_RATES, TIMEOUT_FETCH_CURRENCY_RATES);
		}

	}
}
```
</div></div>

<div class="expand-container">
<a class="expander" href="#">Memory footprint</a>
<div class="expandable-content">

<a href="/images/scheduling_task/nexus5_device/2.png"><img class="caption" src="/images/scheduling_task/nexus5_device/2.png" /></a>
<a href="/images/scheduling_task/nexusS_genymotion/2e.png"><img class="caption" src="/images/scheduling_task/nexusS_genymotion/2e.png" /></a>

</div></div>

**3. ScheduledThreadPoolExecutor**

<div class="expand-container">
<a class="expander" href="#">Expand source code</a>
<div class="expandable-content">
```java	
public class CurrencyManagerScheduledExecutor {
	private static Logger log = LoggerFactory.getLogger(CurrencyManagerScheduledExecutor.class);

	private static final long TIMEOUT_FETCH_CURRENCY_RATES = 1000 * 5; // seconds
	private static long ERROR_BACKOFF_TIMEOUT = TIMEOUT_FETCH_CURRENCY_RATES;
	private static final String PREFS_KEY_CURRENCY_RATES = ".currencyRates";

	private final Context context;
	private ScheduledFuture<?> fetchFuture;

	private static ScheduledThreadPoolExecutor executor;

	public CurrencyManagerScheduledExecutor(@NonNull Context context) {
		this.context = context;
		executor = new ScheduledThreadPoolExecutor(1);

		start();
	}

	private Runnable fetchRunnable = new Runnable() {
		@Override
		public void run() {
			log.debug("Fetching CurrencyRate…");
			RestAdapter restAdapter = new RestAdapter.Builder()
					                          .setEndpoint("http://localhost:8080")
					                          .build();
			CurrencyRatesRestService restService = restAdapter.create(CurrencyRatesRestService.class);
			restService.getRate("USD", currencyRatesRequestCallback);
		}
	};

	private Callback<CurrencyRateResponse.CurrencyRate> currencyRatesRequestCallback = new Callback<CurrencyRateResponse.CurrencyRate>() {
		@Override
		public void success(CurrencyRateResponse.CurrencyRate currencyRate, Response response) {
			ERROR_BACKOFF_TIMEOUT = TIMEOUT_FETCH_CURRENCY_RATES; // default error timeout
			log.info(currencyRate.toString());

			CurrencyRateResponse currencyRateResponse = new CurrencyRateResponse(System.currentTimeMillis(), currencyRate);
			PreferencesToGson.getInstance(context)
			                 .putObject(PREFS_KEY_CURRENCY_RATES, currencyRateResponse);
			EventBus.getDefault().postSticky(currencyRateResponse);
		}

		@Override
		public void failure(RetrofitError error) {
			log.error("Error while fetching currency rates.", error);

			// cancel all subsequent fixed-rate tasks because of an error
			fetchFuture.cancel(true);
			// schedule next fetch with some backoff delay.
			// if error disappears, tasks will be fixed-rate again with "period" parameter
			fetchFuture = executor.scheduleAtFixedRate(fetchRunnable,
			                                           ERROR_BACKOFF_TIMEOUT *= 2,    // backoff delay
			                                           TIMEOUT_FETCH_CURRENCY_RATES,  // period
			                                           TimeUnit.MILLISECONDS);
		}
	};


	private void start() {
		log.debug("Manager started.");

		// First, check if there is sticky event available already
		CurrencyRateResponse currencyRateResponse = EventBus.getDefault().getStickyEvent(CurrencyRateResponse.class);
		if (currencyRateResponse == null) {
			// then check if we have it stored in SharedPreferences
			currencyRateResponse = PreferencesToGson.getInstance(context)
			                                        .getObject(PREFS_KEY_CURRENCY_RATES, CurrencyRateResponse.class);
			if (currencyRateResponse == null) {
				// app fresh start, go fetch rates now
				fetchFuture = executor.scheduleAtFixedRate(fetchRunnable, 0, TIMEOUT_FETCH_CURRENCY_RATES, TimeUnit.MILLISECONDS);
			} else {
				// found in SharedPreferences; post event and schedule fetch meanwhile to update cache
				EventBus.getDefault().postSticky(currencyRateResponse);
				fetchFuture = executor.scheduleAtFixedRate(fetchRunnable, 0, TIMEOUT_FETCH_CURRENCY_RATES, TimeUnit.MILLISECONDS);
			}
		} else {
			// sticky event is there, schedule next fetch
			fetchFuture = executor.scheduleAtFixedRate(fetchRunnable, TIMEOUT_FETCH_CURRENCY_RATES, TIMEOUT_FETCH_CURRENCY_RATES,
			                                           TimeUnit.MILLISECONDS);
		}

	}
}
```
</div></div>

<div class="expand-container">
<a class="expander" href="#">Memory footprint</a>
<div class="expandable-content">

<a href="/images/scheduling_task/nexus5_device/3.png"><img class="caption" src="/images/scheduling_task/nexus5_device/3.png" /></a>
<a href="/images/scheduling_task/nexusS_genymotion/3e.png"><img class="caption" src="/images/scheduling_task/nexusS_genymotion/3e.png" /></a>

</div></div>

**4. AlarmManager**
From the [AlarmManager](http://developer.android.com/reference/android/app/AlarmManager.html) doc:  
Note: The Alarm Manager is intended for cases where you want to have your application code run at a specific time, even if your application is not currently running. For normal timing operations (ticks, timeouts, etc) it is easier and much more efficient to use Handler.

<div class="expand-container">
<a class="expander" href="#">Expand source code</a>
<div class="expandable-content">
```java	
public class CurrencyManagerAlarm {
	private static Logger log = LoggerFactory.getLogger(CurrencyManagerAlarm.class);

	private static final long TIMEOUT_FETCH_CURRENCY_RATES = 1000 * 5; // seconds
	private static long ERROR_BACKOFF_TIMEOUT = TIMEOUT_FETCH_CURRENCY_RATES;
	private static final String PREFS_KEY_CURRENCY_RATES = ".currencyRates";

	private static final String intentFilter = ".managers.sendSchedule";
	private final Context context;
	private final PendingIntent pendingIntent;

	private static AlarmManager alarmManager;

	public CurrencyManagerAlarm(@NonNull Context context) {
		this.context = context;
		alarmManager = (AlarmManager) context.getSystemService(Context.ALARM_SERVICE);

		Intent alarmIntent = new Intent(intentFilter);
		pendingIntent = PendingIntent.getBroadcast(context, 0, alarmIntent, PendingIntent.FLAG_UPDATE_CURRENT);

		FetchCommandReceiver fetchCommandReceiver = new FetchCommandReceiver();
		context.registerReceiver(fetchCommandReceiver, new IntentFilter(intentFilter));

		start();
	}

	private Runnable fetchRunnable = new Runnable() {
		@Override
		public void run() {
			log.debug("Fetching CurrencyRate…");
			RestAdapter restAdapter = new RestAdapter.Builder()
					                          .setEndpoint("http://localhost:8080")
					                          .build();
			CurrencyRatesRestService restService = restAdapter.create(CurrencyRatesRestService.class);
			restService.getRate("USD", currencyRatesRequestCallback);
		}
	};

	private Callback<CurrencyRateResponse.CurrencyRate> currencyRatesRequestCallback = new Callback<CurrencyRateResponse.CurrencyRate>() {
		@Override
		public void success(CurrencyRateResponse.CurrencyRate currencyRate, Response response) {
			ERROR_BACKOFF_TIMEOUT = TIMEOUT_FETCH_CURRENCY_RATES;
			log.info(currencyRate.toString());

			CurrencyRateResponse currencyRateResponse = new CurrencyRateResponse(System.currentTimeMillis(), currencyRate);
			PreferencesToGson.getInstance(context)
			                 .putObject(PREFS_KEY_CURRENCY_RATES, currencyRateResponse);
			EventBus.getDefault().postSticky(currencyRateResponse);
		}

		@Override
		public void failure(RetrofitError error) {
			log.error("Error while fetching currency rates.", error);

			// Cancel all subsequent fixed-rate tasks because of an error.
			// Schedule next fetch with some backoff delay.
			// If error disappears, tasks will be fixed-rate again with "period" parameter
			alarmManager.cancel(pendingIntent);

			long now = Calendar.getInstance().getTimeInMillis();
			alarmManager.setInexactRepeating(AlarmManager.RTC,
			                                 now + (ERROR_BACKOFF_TIMEOUT *= 2),
			                                 TIMEOUT_FETCH_CURRENCY_RATES,
			                                 pendingIntent);
		}
	};


	private void start() {
		log.debug("Manager started.");

		long now = Calendar.getInstance().getTimeInMillis();

		// firstly, check if there is sticky event available already
		// (service may be restarted by system occasionally)
		CurrencyRateResponse currencyRateResponse = EventBus.getDefault().getStickyEvent(CurrencyRateResponse.class);
		if (currencyRateResponse == null) {
			// then check if we have it stored in SharedPreferences
			currencyRateResponse = PreferencesToGson.getInstance(context)
			                                        .getObject(PREFS_KEY_CURRENCY_RATES, CurrencyRateResponse.class);
			if (currencyRateResponse == null) {
				// app fresh start, go fetch rates now
				alarmManager.setRepeating(AlarmManager.RTC,
				                          now,
				                          TIMEOUT_FETCH_CURRENCY_RATES,
				                          pendingIntent);

			} else {
				// found in SharedPreferences; post event and schedule fetch meanwhile to update cache
				EventBus.getDefault().postSticky(currencyRateResponse);
				alarmManager.setRepeating(AlarmManager.RTC,
				                          now,
				                          TIMEOUT_FETCH_CURRENCY_RATES,
				                          pendingIntent);
			}
		} else {
			// sticky event is there, schedule next fetch
			alarmManager.setRepeating(AlarmManager.RTC,
			                          now + TIMEOUT_FETCH_CURRENCY_RATES,
			                          TIMEOUT_FETCH_CURRENCY_RATES,
			                          pendingIntent);
		}

	}

	public class FetchCommandReceiver extends BroadcastReceiver {

		@Override
		public void onReceive(Context context, Intent intent) {
			new Thread(fetchRunnable).start();
		}
	}
}
```
</div></div>

<div class="expand-container">
<a class="expander" href="#">Memory footprint</a>
<div class="expandable-content">

<a href="/images/scheduling_task/nexus5_device/4.png"><img class="caption" src="/images/scheduling_task/nexus5_device/4.png" /></a>
<a href="/images/scheduling_task/nexusS_genymotion/4e.png"><img class="caption" src="/images/scheduling_task/nexusS_genymotion/4e.png" /></a>

</div></div>