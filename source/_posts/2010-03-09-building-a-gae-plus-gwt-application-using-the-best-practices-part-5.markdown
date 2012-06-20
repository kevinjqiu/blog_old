---
layout: post
title: "Building a GAE+GWT application using the best practices (Part 5)"
date: 2012-06-20 13:24
comments: true
categories: java, gwt, gae
---

In the last post of the series, I've shown how to setup a client side GIN+MVP project. This post will discuss the actual building of the UI and GWT-RPC web service calls using GWT-Dispatch on the client side.

## Use cases

Before we dive into the code, let's go over again the use cases our little app has.
<a href="http://reminiscential.files.wordpress.com/2010/02/screenshot2.png"><img src="http://reminiscential.files.wordpress.com/2010/02/screenshot2.png" alt="" title="RateChecker" width="259" height="216" class="aligncenter size-full wp-image-145" /></a>

* When the UI loads, it should automatically fetch the latest saved rate
* Also, the table below the latest rate should display the 10 most recent rates stored in the data store
* The "Refresh" button does the above two steps
* The "Fetch latest" button fetches the rate from the bank website, store it in the data store, and insert the latest rate in the recent rate table

## MVP pattern

For traditional desktop application, there's the MVC (Model-View-Control) pattern that's been in existence for over 20 years, since the Smalltalk days. However, the responsibility separation between the view and controller hasn't been well defined and over the years, people have been arguing about what whether the business logic should be entirely in controller.

With the MVP (Model-View-Presenter) pattern, the view itself doesn't contain any logic. The presenter manipulates the view according to business logic. Therefore, the pattern is also called "supervising controller" or "passive view". This brings a huge benefit because now unit tests only have to deal with presenters, and mock out the view through the display interface. With this setup, the presenter unit tests can be run entirely in JVM. Otherwise, the tests need to involve GWT widgets, which can only be tested with GWTTestCase, which takes a lot longer to run.

## MainView

Here I'm using the new GWT2 UiBinder feature. UiBinder makes developing UI less boring. 

```xml MainView.ui.xml
<!DOCTYPE ui:UiBinder SYSTEM "http://dl.google.com/gwt/DTD/xhtml.ent">
<ui:UiBinder xmlns:ui="urn:ui:com.google.gwt.uibinder"
    xmlns:g="urn:import:com.google.gwt.user.client.ui">
    <ui:style>
        .rateDisplay {
            font-size: x-large;
            font-weight: bolder;
        }
        
        .mainPanel {
            padding: 10px;
        }
    </ui:style>

    <ui:with type="ratechecker.client.bundles.RateCheckerClientBundle"
        field="bundle"></ui:with>
    <g:DecoratorPanel height="200px">
        <g:VerticalPanel styleName="{style.mainPanel}" verticalAlignment="ALIGN_TOP">
            <g:Label>Latest Selling Rate</g:Label>
            <g:Image resource="{bundle.loading}" visible="false"
                ui:field="loading"></g:Image>
            <g:Label ui:field="rateDisplay" styleName="{style.rateDisplay}"></g:Label>
            <g:HorizontalPanel>
                <g:Button ui:field="fetchLatest">Fetch Latest</g:Button>
                <g:Button ui:field="refresh">Refresh</g:Button>
            </g:HorizontalPanel>

            <g:Label>Recent rates</g:Label>
            <g:FlexTable ui:field="rateTable">
            </g:FlexTable>
        </g:VerticalPanel>
    </g:DecoratorPanel>
</ui:UiBinder> 
```

UiBinder can inter-operate with GWT2 ClientBundle. If you have used GWT1.x's ImageBundle, ClientBundle is similar to that, except now with ClientBundle, other client resources are able to be bundled such as CSS stylesheet and external text resource.

### MainPresenter.Display
The display interface is the only thing presenter knows about the UI, and the presenter operates/manipulates UI only through the display interface.

The display interface can be standalone, but I find it's much more convenient to have it as an inner interface inside the presenter class. 
```java
    public interface Display extends WidgetDisplay {
        HasText getRateDisplayLabel();
        HasClickHandlers getFetchLatest();
        HasClickHandlers getRefresh();

        void setEnabledFetchLatestButton(boolean isEnabled);
        void setShowLoadingCurrentRate(boolean isLoading);
        /**
         * Add the rate to the recent rate table.
         * @param rate
         *      The {@link Rate} object
         * @param toHead
         *      <code>true</code> - rate is inserted to the beginning of the table
         *      <code>false</code> - rate is appended to the end of the table
         */
        void addToRecentRates(Rate rate, boolean toHead);
        /**
         * Clear the recent rates table.
         */
        void clearRecentRates();
    }
```

Here we use the "characteristic interface" of the UI elements as return type as they can be mocked. For things that cannot be returned as characteristic interfaces (like FlexTable), we provide methods for the presenter to manipulate the state of the UI objects (such as clearRecentRates()).

### MainView.java

Now we have the display interface, we can map these interface methods onto our UI.
```java MainView.java
package ratechecker.client.mvp;

public class MainView extends Composite implements MainPresenter.Display {

    private static MainViewUiBinder uiBinder = GWT
    .create(MainViewUiBinder.class);

    interface MainViewUiBinder extends UiBinder<Widget, MainView> {
    }

    @UiField
    Button refresh;

    @UiField
    Button fetchLatest;

    @UiField
    Label rateDisplay;

    @UiField
    FlexTable rateTable;

    @UiField
    Image loading;

    private final DateTimeFormat _dateTimeFormat;

    @Inject
    public MainView(final DateTimeFormat dateTimeFormat) {
        _dateTimeFormat = dateTimeFormat;
        initWidget(uiBinder.createAndBindUi(this));
    }

    @Override
    public HasClickHandlers getFetchLatest() {
        return fetchLatest;
    }

    @Override
    public Widget asWidget() {
        return this;
    }

    @Override
    public HasText getRateDisplayLabel() {
        return rateDisplay;
    }

    @Override
    public void setEnabledFetchLatestButton(final boolean isEnabled) {
        fetchLatest.setEnabled(isEnabled);
    }

    @Override
    public void addToRecentRates(final Rate rate, final boolean toHead) {
        final int newRowIdx = toHead ? 0 : rateTable.getRowCount();
        rateTable.insertRow(newRowIdx);
        rateTable.setText(newRowIdx, 0, _dateTimeFormat.format(rate.getTimeFetched()));
        rateTable.setText(newRowIdx, 1, String.valueOf(rate.getRate()));
    }

    @Override
    public void clearRecentRates() {
        rateTable.removeAllRows();
    }

    @Override
    public HasClickHandlers getRefresh() {
        return refresh;
    }

    @Override
    public void setShowLoadingCurrentRate(final boolean isLoading) {
        loading.setVisible(isLoading);
        rateDisplay.setVisible(!isLoading);
    }

    @Override public void startProcessing() { }

    @Override public void stopProcessing() { }
}
```

A lot of this is boilerplate code to satisfy both UiBinder and GWT-presenter.Display interface. Ideally, the VIew shouldn't do too much, if any at all. Realistically, this is harder to achieve.

### MainPresenter

Finally, we can show you the presenter code.
```java MainPresenter.java

public class MainPresenter extends WidgetPresenter<MainPresenter.Display> {

    private final DispatchAsync _dispatch;

    private final ILog _logger;

    @Inject
    public MainPresenter(final Display display, final EventBus eventBus, final DispatchAsync dispatch, final ILog logger) {
        super(display, eventBus);
        _dispatch = dispatch;
        _logger = logger;
    }

    @Override
    protected void onBind() {
        registerHandler(display.getFetchLatest().addClickHandler(new ClickHandler() {

            @Override
            public void onClick(final ClickEvent event) {
                fetchSellingRate();
            }

        }));

        registerHandler(eventBus.addHandler(RateFetchedEvent.TYPE, new RateFetchedHandler() {

            @Override
            public void onRateFetched(final Rate rate) {
                saveRate(rate);
            }

        }));

        registerHandler(eventBus.addHandler(RateSavedEvent.TYPE, new RateSavedHandler() {

            @Override
            public void onRateSaved(final Rate rate) {
                display.addToRecentRates(rate, true);
            }

        }));

        registerHandler(display.getRefresh().addClickHandler(new ClickHandler() {

            @Override
            public void onClick(final ClickEvent event) {
                getLatestSavedRates();
            }

        }));

        getLatestSavedRates();
    }

    void getLatestSavedRates() {
        display.setShowLoadingCurrentRate(true);

        final GetRates getRates = new GetRates();

        _dispatch.execute(getRates, new AsyncCallback<GetRatesResult>() {

            @Override
            public void onFailure(final Throwable caught) {
                display.setShowLoadingCurrentRate(false);
                _logger.error("Unable to get saved rates: " + caught.getMessage());
            }

            @Override
            public void onSuccess(final GetRatesResult result) {
                display.setShowLoadingCurrentRate(false);
                display.clearRecentRates();

                for (final Rate rate : result.getRates()) {
                    display.addToRecentRates(rate, true);
                }

                // Put the latest rate in the box
                if (result.getRates().size() > 0) {
                    final Rate latestRate = result.getRates().get(0);
                    display.getRateDisplayLabel().setText(String.valueOf(latestRate.getRate()));
                }
            }
        });

    }

    void fetchSellingRate() {
        display.setShowLoadingCurrentRate(true);
        final CheckRate checkRate = new CheckRate(RateType.Selling);
        _dispatch.execute(checkRate, new AsyncCallback<CheckRateResult>() {

            @Override
            public void onFailure(final Throwable caught) {
                display.setShowLoadingCurrentRate(false);
                _logger.error("Unable to fetch rate: " + caught.getMessage());
            }

            @Override
            public void onSuccess(final CheckRateResult result) {
                display.setShowLoadingCurrentRate(false);
                // enable the fetch button
                display.setEnabledFetchLatestButton(true);
                display.getRateDisplayLabel().setText(String.valueOf(result.getRate().getRate()));
                eventBus.fireEvent(new RateFetchedEvent(result.getRate()));
            }

        });

        // disable the fetch button until RPC succeeds
        display.setEnabledFetchLatestButton(false);
    }

    void saveRate(final Rate rate) {
        final SaveRate saveRate = new SaveRate(rate);

        _dispatch.execute(saveRate, new AsyncCallback<SaveRateResult>() {

            @Override
            public void onFailure(final Throwable caught) {
                _logger.error("Unable to save rate: " + caught.getMessage());
            }

            @Override
            public void onSuccess(final SaveRateResult result) {
                eventBus.fireEvent(new RateSavedEvent(rate));
            }

        });

    }

    @Override protected void onPlaceRequest(final PlaceRequest request) { }

    @Override protected void onUnbind() {}

    @Override public void refreshDisplay() {}

    @Override public void revealDisplay() {}

    @Override public Place getPlace() { return null; }
}
```

In the binding process, the event handlers are attached to the view components. `MainPresenter.bind()` was explicitly called by `AppPresenter.go()`. This is a simple application with one presenter. If there are more presenters, `AppPresenter` needs to manage the state of these sub-presenters: if they're active, the `bind()` method is called. If the presenter is no-longer active, the presenter's `unbind()` method should be called to un-attach the handlers, so they don't interfere with the event handlers that are currently in the active presenter.

The presenter is also responsible for making web service calls and deal with the returns. To call GWT-RPC web service using GWT-dispatch, we inject a DispatchAsync, which is an asynchronous counter part of the DispatchServlet introduced a few posts ago.

To call a web service, we simply construct an action object with required parameters and pass it in `DispatchAsync.execute()` and expect an `AsyncCallback` of type result that's coupled with the action. (remember each action has a coupled result type). Also, in this application, every action has a related event to indicate whether the action is successful. The event is thrown onto the event bus, so any interested party can handle that. The main benefit of using event bus is that my web service calls don't have to be coupled with the subsequent actions. For example, saveRate() method is responsible for making the web service calls, but the subsequent action (adding the saved rate to the recent rate table) isn't part of saveRate() method, and it shouldn't be. If in the future, some other actions need to be carried out when a rate is saved, we just have to add the action in the RateSavedHandler, and indeed, if another part of the UI (not visible by main presenter) need to do something after the rate is saved, that presenter only needs to handle that event in there without affecting saveRate() method at all.

For view the full source code, take a look at the project I created on [Github](http://github.com/kevinjqiu/ratechecker).

**EDIT:** For any Google App Engine experts out there happened to be reading this post, I'm having trouble with the performance of this simple app. Seems like the data store is taking way too much time executing my query. Initially I thought it was because URL fetch is slow, but I recently added a property in Rate entity to track the time spent on fetching the URL and every request takes less than 1 second. However, the GetRates action takes a long time to return (usually ~3 to 5 seconds, sometimes even over 10 seconds). It's a simple query ordering on a single property so no complex index is needed. So I'm wondering what's wrong here.

