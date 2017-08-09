---
title: egui Documentation

language_tabs: # must be one of https://git.io/vQNgJ
  - java

includes:

search: true
---

# Introduction

Welcome to the Thermo Fisher Scientific egui Library! You can use our library to use a multitude of functions originating from the Monarch project. Below are the descriptions of the various functions available, as well as some code examples in the right column.

## Import from Local File

1. Go to File>New>New Module
2. Select "Import .JAR/.AAR Package" and click next.
3. Enter the path to .aar file and click finish.
4. Go to File>Project Structure (Ctrl+Shift+Alt+S).
5. Under "Modules," in left menu, select "app."
6. Go to "Dependencies tab.
7. Click the green "+" in the upper right corner.
8. Select "Module Dependency"
9. Select the new module from the list.

## Import from Github 

Go to JitPack.com. 

Type up repo in Look Up 

Use latest release to see what to add

Public Repo: 

1. Go to root (not app) build.gradle. Add maven { url 'https://jitpack.io' } in all projects { repositories … }
2. Go to app/build.gradle. Add compile 'com.github.bhuangy:egui:0.1.0'
3. These instructions are automatically generated when you look up the repo on JitPack

Private Repo:

1. Must first authorize the private repo
2. Add token   authToken=d3ba024cb9693c92d7ec65a78ef0b52e15231f06  in root gradle.properties
3. Paste code as shown on JitPack
4. Add code for app/build.gradle as seen in Public Repo instructions
5. Instructions are on JitPack as well


# Utilities

## Logger

> Logger Example - This will output to the specified file in logback.xml

```java
import com.example.egui.Utilities.logger.DCLog;

private static final DCLog LOG = DCLog.getLogger(MainActivity.class);

@Override
protected void onCreate(Bundle savedInstanceState) {
    LOG.d("This is an example of how to use the logger");
}
```

Allows user to add log statements, which are then sent to an output file that the user can specify. The log statements are labelled with the 

### Instructions to setup

You must import logback.xml into app/src/main/assets - The correct file can be found in library code.

Go to logback.xml and set the LOG_PATH value to the directory you want to get the log file from. Use the getCacheDir().toString() function in order to see which directory to set LOG_PATH to.


Change logger_name=“…” to your current package name. To access with permissions - go to your sdk directory. In order to see your sdk directory, go to the sdk manager in Android Studio and copy the file path in “Android SDK Location”. Locally, go to platform-tools. Run ./adb shell. Run “run-as com.example” replacing com.example with your package name. Go to the cache dir, and the log file will be in there


### Types of Log functions

Type | Description
--------- | -----------
e | error
w | warn
i | info
d | debug
v | trace


## SCPI

> Thread Method: This is a simplified code example, containing only certain parts of the application that are required for the SCPI client to start and send commands. It is recommended to create your own application and only use this code as a reference.

```java

String ipAddress = "10.0.2.2";
int port = 7000;
QueryVersion queryVersion = new QueryVersion();
ScpiCommand state = new State("Cartridge:Inserted", true);
Button b1;

public class MainActivity extends Activity {

  protected void onCreate(Bundle savedInstanceState) {

    final Intent ISCommandIntent = new Intent(this, ISCommandService.class);
    final Intent broadcastIntent = new Intent(this, BroadcastActivity.class);

    /* ThreadHandler */
    final Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            Bundle bundle = msg.getData();
            boolean serviceStarted = bundle.getBoolean("myKey");

            if (serviceStarted) {
                Toast.makeText(getApplicationContext(), "Service is Ready: Sent Command",Toast.LENGTH_SHORT).show();
                ISCommandService.sendCommand(state);

            } else {
                Toast.makeText(getApplicationContext(), "Thread: Service Is Not Ready Yet",Toast.LENGTH_SHORT).show();
            }
        }
    };

    /* First thread to start service */
    Runnable runnable = new Runnable() {
         @Override
         public void run() {

             /* Create wait with 20 seconds to stall service start */
             long endTime = System.currentTimeMillis()
                     + 20*1000;

             while (System.currentTimeMillis() < endTime) {
                 synchronized (this) {
                     try {
                         wait(endTime -
                                 System.currentTimeMillis());
                     } catch (Exception e) {}
                 }
             }

             startService(ISCommandIntent);
         }
    };

    Thread serviceStart = new Thread(runnable);
    serviceStart.start();

    /* Button */
    b1.setOnClickListener(new View.OnClickListener() {
      @Override
      public void onClick(View v) {

        /* Second Thread to check if client is ready and to send command if it is */
        Runnable scpiMessenger = new Runnable() {
          @Override
          public void run() {
              Message msg = handler.obtainMessage();
              Bundle bundle = new Bundle();

              LOG.d("isInstrumentServerReady value: " + String.valueOf(ISCommandService.isInstrumentServerReady));
              bundle.putBoolean("myKey", ISCommandService.isInstrumentServerReady);
              msg.setData(bundle);

              if (ISCommandService.isInstrumentServerReady) {
                  ISCommandService.sendCommand(state);

              } else {

                  LOG.d("Thread: Service Is Not Ready Yet");
              }
              handler.sendMessage(msg);
          }
        };

        Thread scpiMessage = new Thread(scpiMessenger);
        scpiMessage.start();

    }

  }

  Thread scpiMessage = new Thread(scpiMessenger);
  scpiMessage.start();

```

Communicates with the Instrument Server. i.e. sends commands, receives responses, controls the physical instrument.

### Instructions to Setup

<aside class="warning"> These instructions are for testing on the Instrument Server on a local machine </aside>

Run ./instrument.py (sudo ./instrument.py if permissions are denied)

In a new terminal, run telnet localhost 2323 - this will allow you to send commands to the Instrument Server to test functionality. 

Check code example for Android functionality.

### Thread Method Code explanation

This uses Threads to separately control when the scpi starts and when it sends commands in order to avoid a situation in which the scpi will send a command when it has not started yet.

The ipAddress is defined for the local machine host, in this case it is 10.0.2.2. The port is pre-defined as 7000

The Scpi Command state will send the command to the Instrument Server telling it to change its state to cartridge:inserted true.

Create the Thread Handler to receive information from the Threads that we will use. Use the handler to interact with the Instrument Server i.e. send commands and alter the state of the instrument. This allows the SCPI client to only send commands once it has been started, and is ready to send and receive information.

The first Thread is used to create a delay before starting the scpi service. This is for testing purposes, to ensure the scpi client will not try to send a command before it is ready.

The Thread that is attached to the Button will check if the scpi client is ready before sending a message to the Handler to initiate the command. This occurs upon the button being clicked.

> Broadcast Sender/Receiver Method: This example shows how to use the scpi client using broadcast receivers. The ability to send broadcasts was added to the ISCommandService, which allows the client to send a signal saying that it will start. Whoever is listening using a broadcast receiver will receive that signal and initiate commands, ensuring that it will only act once the client is ready.

```java

public class BroadcastActivity extends Activity {

    String ipAddress = "10.0.2.2";
    int port = 7000;
    private static final String SHARED_SECRET_IS = "MbmfSAtNA4f";
    private static final DCLog LOG = DCLog.getLogger(BroadcastActivity.class);
    ScpiCommand state = new State("Cartridge:Inserted", true);
    Button commandButton, b4;


    private BroadcastReceiver scpiReadyReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            commandButton = findViewById(R.id.commandButton);


            if (intent.getAction().equals(SuperService.SCPI_READY_INTENT)) {

                ISCommandService.setAccessLevel(Access.Level.Controller, SHARED_SECRET_IS);
                ISCommandService.sendCommand(state);
                ISCommandService.getVersion("InstrumentServer:Version");
                LOG.e("Received Broadcast");

            } else if (intent.getAction().equals(SuperService.SCPI_NOT_READY_INTENT)) {
                LOG.e("ISMessageService Scpi is NOT ready");
            }

            commandButton.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View view) {
                    ISCommandService.setAccessLevel(Access.Level.Administrator, SHARED_SECRET_IS);
                    ISCommandService.sendCommand(state);
                    ISCommandService.getVersion("InstrumentServer:Version");
                }
            });


        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_broadcast);


        /* Set to 10.0.2.2 - Instrument Server ipAddress */
        ISCommandService.setServerIpAddress(ipAddress, port);
        b4 = findViewById(R.id.activityButton);


        /* Start Service to broadcast SCPI is ready */
        final Intent ISCommandIntent = new Intent(this, ISCommandService.class);
        final Intent mainIntent = new Intent(this, MainActivity.class);

        /* Register Receiver */
        LOG.d("Register Broadcast Receiver");
        registerReceiver(scpiReadyReceiver, new IntentFilter(SuperService.SCPI_READY_INTENT));
        registerReceiver(scpiReadyReceiver, new IntentFilter(SuperService.SCPI_NOT_READY_INTENT));
        registerReceiver(scpiReadyReceiver, new IntentFilter(SuperService.TOUCH_INTENT));

        startService(ISCommandIntent);

    }

```

### Broadcast Send/Receiver Method Code Explanation

First create the BroadcastReceiver. The SuperService.SCPI_READY_INTENT is a predefined broadcast that is sent out once the scpi client is initialized to start. If the receiver receives the correct broadcast, send the commands to the Instrument Server.

In onCreate, register the receivers, and start the Service, which should send the broadcast to the receivers.

# Widgets

UI Widgets that allow for common UX solutions and themes. 

> Widgets Code Example

```java
package com.thermofisher.eguiexample;

import android.app.Activity;
import android.os.Bundle;
import android.os.Handler;
import android.view.View;
import android.widget.LinearLayout;
import android.widget.TextView;

import com.example.egui.Widgets.BaseDialogFragment;
import com.example.egui.Widgets.ConfirmDialogFragment;
import com.example.egui.Widgets.EditTextDialogFragment;
import com.example.egui.Widgets.EnterPinDialogFragment;
import com.example.egui.Widgets.NavigationBar;
import com.example.egui.Widgets.NumberPad;
import com.example.egui.Widgets.ProgressFragment;
import com.example.egui.Widgets.TableView;

import java.util.ArrayList;
import java.util.List;

import static android.text.InputType.TYPE_CLASS_TEXT;
import static android.text.InputType.TYPE_TEXT_VARIATION_PERSON_NAME;

/**
 * Created by brandon.huang on 7/31/17.
 */

public class NavigationBarActivity extends Activity {

    private NavigationBar navigationBar;
    private LinearLayout page1, page2, page3, page4, page5;
    private NumberPad numberPad;
    private TableView tableView;
    private int count;
    private TextView textView, numberPadText, confirmText;
    private ProgressFragment progressDialog;



    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.nav_bar);

        // Initialize navigationBar
        navigationBar = (NavigationBar) findViewById(R.id.navigationBar);
        navigationBar.addItem("Table");
        navigationBar.addItem("Keyboard");
        navigationBar.addItem("Number Pad");
        navigationBar.addItem("Confirm");
        navigationBar.addItem("Progress");
        navigationBar.setOnItemClickListener(navigationBarItemClickListener);

        //Initialize LinearLayouts
        page1 = (LinearLayout) findViewById(R.id.Layout1);
        page2 = (LinearLayout) findViewById(R.id.Layout2);
        page3 = (LinearLayout) findViewById(R.id.Layout3);
        page4 = (LinearLayout) findViewById(R.id.Layout4);
        page5 = (LinearLayout) findViewById(R.id.Layout5);


        //Set only page 1 to appear initially
        page2.setVisibility(View.GONE);
        page3.setVisibility(View.GONE);
        page4.setVisibility(View.GONE);
        page5.setVisibility(View.GONE);

        //initialize layout parts
        tableView = (TableView) findViewById(R.id.tableView);
        textView = (TextView) findViewById(R.id.textView);
        numberPadText = (TextView) findViewById(R.id.numberPadText);
        confirmText = (TextView) findViewById(R.id.confirmText);


        List<String> tableHeader = new ArrayList<>();
        tableHeader.add("Column 1");
        tableHeader.add("Column 2");
        tableHeader.add("Column 3");

        List<List<String>> profilesData = new ArrayList<>();

        for (int i = 0; i < 5; i++) {
            List<String> profileData = new ArrayList<>();

            for (int j = 0; j < 3; j++) {
                profileData.add(Integer.toString(count++));
            }
            profilesData.add(profileData);
        }

        tableView.setColumnWidths(new int[] {150, 300, 300});
        tableView.setColumnHeaders(tableHeader);
        tableView.setData(profilesData);

        textView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {

                EditTextDialogFragment fragment = new EditTextDialogFragment();
                fragment.setCanBeEmpty(false);
                fragment.setTitle("egui Example");
                fragment.setTextViewToEdit(textView);
                fragment.setInputType(TYPE_CLASS_TEXT | TYPE_TEXT_VARIATION_PERSON_NAME);
                fragment.show(getFragmentManager(), "editTextDialog");

            }
        });

        numberPadText.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                final EnterPinDialogFragment pinFragment = new EnterPinDialogFragment();
                //NumberPadDialogFragment fragment = new NumberPadDialogFragment();
                pinFragment.setTitle("Enter Pin");
                pinFragment.show(getFragmentManager(), "editTextDialog");
                final ConfirmDialogFragment confirmPin = new ConfirmDialogFragment();
                confirmPin.showSecondaryButton(false);

                pinFragment.setDismissListener(new BaseDialogFragment.DismissListener() {
                    @Override
                    public void onDismiss(Bundle bundle) {
                        if (pinFragment.getPinEntered().equals("3333")) {
                            confirmPin.setPrimaryButtonText("OK");
                            confirmPin.show(getFragmentManager(), "Pin Confirmation");
                            confirmPin.setPrimaryMessage("Correct Pin");
                        } else {
                            confirmPin.setPrimaryButtonText("OK");
                            confirmPin.show(getFragmentManager(), "Pin Confirmation");
                            confirmPin.setPrimaryMessage("Incorrect Pin \nEntered: " + pinFragment.getPinEntered());
                        }
                    }
                });
            }
        });

        confirmText.setText("Confirm");
        confirmText.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                ConfirmDialogFragment fragment = new ConfirmDialogFragment();
                fragment.setPrimaryButtonText("Primary Button");
                fragment.setSecondaryButtonText("Secondary Button");
                fragment.show(getFragmentManager(), "Confirm");
                fragment.setPrimaryMessage("Confirmation");
            }
        });

    }

    private NavigationBar.OnItemClickListener navigationBarItemClickListener = new NavigationBar.OnItemClickListener() {

        @Override
        public void onItemClicked(View View, int itemIndex) {
            if (itemIndex == 0) {
                page1.setVisibility(android.view.View.VISIBLE);
                page2.setVisibility(View.GONE);
                page3.setVisibility(View.GONE);
                page4.setVisibility(View.GONE);
                page5.setVisibility(View.GONE);
            } else if (itemIndex == 1){
                page1.setVisibility(View.GONE);
                page2.setVisibility(android.view.View.VISIBLE);
                page3.setVisibility(View.GONE);
                page4.setVisibility(View.GONE);
                page5.setVisibility(View.GONE);

            } else if (itemIndex == 2) {
                page1.setVisibility(View.GONE);
                page2.setVisibility(View.GONE);
                page3.setVisibility(android.view.View.VISIBLE);
                page4.setVisibility(View.GONE);
                page5.setVisibility(View.GONE);

            } else if (itemIndex == 3){
                page1.setVisibility(View.GONE);
                page2.setVisibility(View.GONE);
                page3.setVisibility(View.GONE);
                page4.setVisibility(android.view.View.VISIBLE);
                page5.setVisibility(View.GONE);

            } else {
                page1.setVisibility(View.GONE);
                page2.setVisibility(View.GONE);
                page3.setVisibility(View.GONE);
                page4.setVisibility(View.GONE);
                page5.setVisibility(android.view.View.VISIBLE);

                progressDialog = new ProgressFragment();
                progressDialog.setTitle("egui Test Progress");
                progressDialog.setProgressMessage("Loading");
                progressDialog.show(getFragmentManager(), null);

                final Handler handler = new Handler();
                handler.postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        progressDialog.dismiss();
                    }
                }, 5000);
            }
        }
    };
}
```
### Navigation Bar

The Navigation Bar is used in order to switch tabs between Layouts in app. It can be used in conjunction with other Widgets i.e. have one Widget per tab (Layout). 

### Custom Keyboard

The Custom Keyboard widget is meant to be used as a uniform keyboard for all Thermo Fisher products. It is a pop-up keyboard that appears once the user initiates a certain action i.e. clicking a button or text.

#### Instructions to Setup

In appropriate layout xml file, state CustomKeyboardView. Example code shows how to properly call EditText Fragment in order to edit using the keyboard.

###Table

A table widget with specified columns and rows. The code example creates a 3 columned table, and populates it with numbers.

### Enter Pin

A pin pad created using the numberpad widget. Can register it to check for a correct pin number. The code example uses a textview as a button, which leads to a enter pin Fragment Dialog.

### Confirm Dialog

A confirmation page that has a primary and secondary button. The code example creates a confirmation page with the message set to "Confirmation", and a Primary and Seconday Button.

### Progress Dialog

A progress page that has a splash animation. The code example displays a Progress page that stays on for 5 seconds before being dismissed.

