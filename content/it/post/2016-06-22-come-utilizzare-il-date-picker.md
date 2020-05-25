---
Description: Il DatePicker fornisce un'interfaccia grafica già pronta che gli utenti
  possono utilizzare per selezionare una data. In questo tutorial spiego brevemente
  come utilizzarlo, avvalendoci anche della libreria joda-time.
author: Umberto D'Ovidio
date: "2016-06-22T00:00:00Z"
title: Come Utilizzare il DatePicker in AndroidStudio
---
Il DatePicker permette di scegliere una data costituita da giorno, mese e anno.
In questo post spieghero’ come creare un datepicker dialog.
<!--more-->

Creeremo un’app che, una volta scelta una data, ci dirà quanti millisecondi,
secondi, minuti, ore, giorni, mesi ed anni sono passati dalla data in questione.

Per farlo utilizzeremo una libreria esterna chiamata joda-time, molto utile
per lavorare con le date.

Il layout per la nostra app sarà molto semplice, utilizzeremo un pulsante e una
TextView. Quando il pulsante verrà premuto, l’utente potrà scegliere una data,
e una volta fatto ciò setteremo la nostra textView con i giorni passati dalla
data scelta.

### Layout

{{< highlight xml >}}
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:tools="http://schemas.android.com/tools"
android:layout_width="match_parent"
android:layout_height="match_parent"
android:paddingBottom="@dimen/activity_vertical_margin"
android:paddingLeft="@dimen/activity_horizontal_margin"
android:paddingRight="@dimen/activity_horizontal_margin"
android:paddingTop="@dimen/activity_vertical_margin"
tools:context="com.example.cyborg.giornipassati.MainActivity">

  <Button
  android:layout_width="wrap_content"
  android:layout_height="wrap_content"
  android:text="Scegli una data!"
  android:id="@+id/button"
  android:layout_alignParentBottom="true"
  android:layout_centerHorizontal="true"/>

  <TextView
  android:layout_width="wrap_content"
  android:layout_height="wrap_content"
  android:id="@+id/textView"
  android:layout_centerVertical="true"
  android:layout_centerHorizontal="true"/>
</RelativeLayout>
{{< / highlight >}}


### Main Activity

{{< highlight java >}}

package com.example.cyborg.giornipassati;

import android.app.DialogFragment;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

  private Button mButton;
  private TextView mTextView;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    mButton = (Button) findViewById(R.id.button);
    mTextView = (TextView) findViewById(R.id.textView);

    mButton.setOnClickListener(new View.OnClickListener() {
      @Override
      public void onClick(View v) {
        DialogFragment datePicker = new DatePickerFragment();
        datePicker.show(getFragmentManager(),&quot;Date Picker&quot;);

      }
    });
  }
}
{{< / highlight >}}


### DatePickerFragment

{{< highlight java >}}

package com.example.cyborg.giornipassati;

import android.app.DatePickerDialog;
import android.app.Dialog;
import android.app.DialogFragment;
import android.content.Context;
import android.net.Uri;
import android.os.Bundle;
import android.support.v4.app.Fragment;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.Button;
import android.widget.DatePicker;
import android.widget.TextView;

import org.joda.time.DateTime;
import org.joda.time.Days;
import org.joda.time.Minutes;
import org.joda.time.Months;
import org.joda.time.Seconds;
import org.joda.time.Years;

import java.util.Calendar;
import java.util.Date;

public class DatePickerFragment extends DialogFragment implements DatePickerDialog.OnDateSetListener {

  @Override
  public Dialog onCreateDialog(Bundle savedInstanceState){
    //Use the current date as the default date in the date picker
    final Calendar c = Calendar.getInstance();
    int year = c.get(Calendar.YEAR);
    int month = c.get(Calendar.MONTH);
    int day = c.get(Calendar.DAY_OF_MONTH);
    return new DatePickerDialog(getActivity(), this, year, month, day);
  }

  public void onDateSet(DatePicker view, int year, int month, int day) {
    //Do something with the date chosen by the user
    Date dataOdierna = new Date();
    DateTime oggi = new DateTime(dataOdierna);
    Calendar calendar = Calendar.getInstance();
    calendar.set(year, month, day);
    Date dataScelta = calendar.getTime();
    DateTime scelta = new DateTime(dataScelta);

    int anni = Years.yearsBetween(scelta, oggi).getYears();
    int mesi = Months.monthsBetween(scelta, oggi).getMonths();
    int giorni = Days.daysBetween(scelta, oggi).getDays();
    int minuti = Minutes.minutesBetween(scelta, oggi).getMinutes();
    int secondi = Seconds.secondsBetween(scelta, oggi).getSeconds();
    long millisecondi = dataOdierna.getTime() - dataScelta.getTime();

    TextView tv = (TextView) getActivity().findViewById(R.id.textView);

    String messaggio = String.format("Sono passati %d anni, %d mesi, %d giorni, %d minuti,  %d secondi",
    anni,
    mesi,
    giorni,
    minuti,
    secondi,
    millisecondi);
    tv.setText(messaggio);
  }
}
{{< / highlight >}}
