# Suitelet
Rendering/Creating Components, External URL / URL

## Overview
Suitelets are server-side scripts used to create custom pages or integrations in NetSuite. They can render HTML, create forms, and interact with external URLs.

---

## Code Snippet: Rendering Components

```javascript
/**
 * @NApiVersion 2.x
 * @NScriptType Suitelet
 */
define(['N/ui/serverWidget', 'N/record'], function(serverWidget, record) {
    function onRequest(context) {
        if (context.request.method === 'GET') {
            // Create a form
            var form = serverWidget.createForm({
                title: 'Custom Suitelet Form'
            });

            // Add fields to the form
            form.addField({
                id: 'custpage_text',
                type: serverWidget.FieldType.TEXT,
                label: 'Enter Text'
            });

            form.addSubmitButton({
                label: 'Submit'
            });

            // Render the form
            context.response.writePage(form);
        } else if (context.request.method === 'POST') {
            // Handle form submission
            var textValue = context.request.parameters.custpage_text;
            context.response.write('You entered: ' + textValue);
        }
    }

    return {
        onRequest: onRequest
    };
});
```
## Code Snippet: Redirecting to an External URL
```javascript
context.response.sendRedirect({
    url: 'https://www.example.com'
});
```
## Code Snippet: Suitelet Rendering a Custom HTML Page with a Table
```javascript
/**
 * @NApiVersion 2.1
 * @NScriptType Suitelet
 */
define(["N/log", "N/query", "N/record", "N/search", "N/ui/serverWidget", "N/runtime", "N/https", "moment"], function (
  log,
  query,
  record,
  search,
  serverWidget,
  runtime,
  https,
  moment
) {
  function onRequest(context) {
    try {
      var response = context.response;
      if (context.request.method === "GET") {
        // Fetch data for the table
        var providersList = providerList("mon"); // Example: Fetch providers available on Monday
        var bookedTime = eventSearch(moment().format("M/D/YYYY")); // Example: Fetch bookings for today
        var exceptionTime = getExceptionTime(moment().format("M/D/YYYY")); // Example: Fetch exceptions for today

        // Generate HTML for the table
        var html = `<!DOCTYPE html>
        <html>
        <head>
            <title>Provider Time Booked</title>
            <link href="https://cdn.jsdelivr.net/npm/select2@4.1.0-rc.0/dist/css/select2.min.css" rel="stylesheet" />
            <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
            <script src="https://cdn.jsdelivr.net/npm/select2@4.1.0-rc.0/dist/js/select2.min.js"></script>
            <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/flatpickr/dist/flatpickr.min.css">
            <script src="https://cdn.jsdelivr.net/npm/flatpickr"></script>
            <style>
                table {
                    width: 100%;
                    border-collapse: collapse;
                    margin-top: 20px;
                }
                th, td {
                    border: 1px solid #ddd;
                    padding: 8px;
                    text-align: left;
                }
                th {
                    background-color: #f2f2f2;
                }
            </style>
        </head>
        <body>
            <div class="container">
                <h3>Provider Time Booked</h3>
                <div class="selectors-container">
                    <div class="selector">
                        <label for="provider-select">Provider:</label>
                        <select id="provider-select" style="width: 100%" multiple>
                            <option value="All">All</option>
                            ${providersList
                              .map(
                                (provider) =>
                                  `<option value="${provider.id}">${provider.firstname} ${provider.lastname}</option>`
                              )
                              .join("")}
                        </select>
                    </div>
                    <div class="selector">
                        <label for="date-select">Select Date:</label>
                        <input type="text" id="date-select" style="width: 100%" placeholder="Select a date...">
                    </div>
                </div>

                <table id="dailyTable">
                    <thead>
                        <tr>
                            <th>Provider</th>
                            <th>Working Time</th>
                            <th>Booked Time</th>
                            <th>Booked %</th>
                        </tr>
                    </thead>
                    <tbody>
                        ${providersList
                          .map((provider) => {
                            const workingTime = parseInt(provider.custentity_bh_wc_mon_av_min || 0);
                            const bookedMinutes = parseInt(bookedTime[provider.id] || 0);
                            const bookingPercentage = workingTime > 0 ? Math.round((bookedMinutes / workingTime) * 100) : 0;

                            return `
                            <tr data-provider-id="${provider.id}">
                                <td>${provider.firstname} ${provider.lastname}</td>
                                <td>${workingTime} minutes</td>
                                <td>${bookedMinutes} minutes</td>
                                <td>${bookingPercentage}%</td>
                            </tr>
                            `;
                          })
                          .join("")}
                    </tbody>
                </table>
            </div>

            <script>
                $(document).ready(function() {
                    $('#provider-select').select2();
                    $('#provider-select').on('change', filterTable);

                    flatpickr("#date-select", {
                        dateFormat: "Y-m-d",
                        defaultDate: new Date(),
                        onChange: function(selectedDates) {
                            refreshTable(selectedDates);
                        }
                    });

                    function refreshTable(selectedDates) {
                        const selectedDate = selectedDates[0];
                        const formattedDate = moment(selectedDate).format('M/D/YYYY');

                        $.ajax({
                            url: window.location.href,
                            data: { date: formattedDate },
                            success: function(response) {
                                const parser = new DOMParser();
                                const doc = parser.parseFromString(response, 'text/html');
                                $('#dailyTable tbody').html($(doc).find('#dailyTable tbody').html());
                                filterTable();
                            }
                        });
                    }

                    function filterTable() {
                        const selectedProviders = $('#provider-select').val();
                        $('tbody tr').each(function() {
                            const row = $(this);
                            const providerId = row.attr('data-provider-id');
                            const showRow = selectedProviders.includes("All") || selectedProviders.includes(providerId);
                            row.toggle(showRow);
                        });
                    }
                });
            </script>
        </body>
        </html>`;

        response.write(html);
      }
    } catch (e) {
      log.error("Error", e);
      response.write(`
                <html>
                    <body>
                        <h2>An error occurred:</h2>
                        <pre>${e.message}</pre>
                    </body>
                </html>
            `);
    }
  }

  function providerList(day) {
    var query = `
        SELECT 
            id,
            firstname,
            lastname,
            custentity_bh_wc_${day}_av_min
        FROM employee
        WHERE cseg_bh_emp_typ_seg = 2
        AND isinactive = 'F'
        ORDER BY firstname, lastname
    `;

    return query.runSuiteQL({ query: query }).asMappedResults();
  }

  function eventSearch(nsDate) {
    var eventsQuery = `
        SELECT 
            custevent_bh_eve_prov as provider_id,
            SUM(custevent_bh_eve_lnth) as total_duration
        FROM calendarevent 
        WHERE TRUNC(startdate) = TO_DATE('${nsDate}', 'MM/DD/YYYY')
        AND cseg_bh_custseg_rzn NOT IN (28, 16)
        AND custevent_bh_eve_app_stat NOT IN (2, 4, 6, 7, 9, 10)
        AND custevent_bh_eve_prov IS NOT NULL
        GROUP BY custevent_bh_eve_prov
        ORDER BY custevent_bh_eve_prov
    `;

    var queryResults = query.runSuiteQL({ query: eventsQuery }).asMappedResults();
    var providerBookings = {};
    queryResults.forEach(function (result) {
      providerBookings[result.provider_id] = result.total_duration || 0;
    });

    return providerBookings;
  }

  function getExceptionTime(nsDate) {
    var eventsQuery = `
        SELECT 
            custevent_bh_eve_prov as provider_id,
            SUM(custevent_bh_eve_lnth) as total_duration
        FROM calendarevent 
        WHERE TRUNC(startdate) = TO_DATE('${nsDate}', 'MM/DD/YYYY')
        AND cseg_bh_custseg_rzn IN (28)
        AND custevent_bh_eve_prov IS NOT NULL
        GROUP BY custevent_bh_eve_prov
        ORDER BY custevent_bh_eve_prov
    `;

    var queryResults = query.runSuiteQL({ query: eventsQuery }).asMappedResults();
    var providerBookings = {};
    queryResults.forEach(function (result) {
      providerBookings[result.provider_id] = result.total_duration || 0;
    });

    return providerBookings;
  }

  return {
    onRequest: onRequest,
  };
});
```
## Key Features of This Snippet

1. **Dynamic HTML Rendering**: The Suitelet generates an HTML page with a table populated by data fetched from NetSuite records.
2. **External Libraries**: Uses jQuery, Select2 (for dropdowns), and Flatpickr (for date pickers) to enhance the UI.
3. **AJAX for Dynamic Updates**: The table can be refreshed dynamically based on user input (e.g., date selection or provider filtering).
4. **Error Handling**: Includes basic error handling to display errors in the UI if something goes wrong.
