# Adding checkboxes to Google sheets, because EVE

This post will be on a specific detail of something I've recently done. I do plan on putting some over-arching design posts up as well soon.

In EVE a lot of things are kept secret for what is called op-sec reasons, and there will be a lot of that in this blog. So I won't go into the why of it.
Another thing abouve EVE Echoes is that I call it the greatest text adventure game ever. As an executor, for better or worse, a great deal of my time spent playing EVE Echoes is spent on Discord. And for a lot of people another part of it is spent in google sheets.
EVE is sometimes referred to as a heavy weight Excel client. Which is why things like this happen: https://arstechnica.com/gaming/2022/05/eve-onlines-ms-excel-partnership-makes-spreadsheets-in-space-official/

And now, with the help of Discord bots, the two can meet. Discord bots will be a common topic here.
The basic thing I needed to do was get a bot to list something for a user. And I realized they were going to just dump it into google sheets anyways. So, it makes sense to just create that sheet.
On top of that, the user would want a column full of checkboxes to go with the data. Now that on its own is pretty easy to create in the UI: 

![image](https://user-images.githubusercontent.com/52060413/195969961-9e0e63e7-ff20-4532-998e-b7dec1d69c31.png)

But it wasn't so obvious from the API perspective. I had to do with any seasoned developer does. Search on google for stackoverflow answers.

Now our bot is in C#, and the google API docs don't cater to C#.
This is the linke I got the most use out of: 
[https://webapps.stackexchange.com/questions/121502/how-to-add-checkboxes-into-cell-using-google-sheets-api-v4](https://webapps.stackexchange.com/questions/121502/how-to-add-checkboxes-into-cell-using-google-sheets-api-v4).

A careful inspection of this will help once you know that you need a batch update for a conditional formatting request: 
[https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets/request](https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets/request)

And this is what the C# code looks like: 

```C#
            BatchUpdateSpreadsheetRequest request = new BatchUpdateSpreadsheetRequest
            {
                Requests = new List<Request>(
                new[]
                {
                    new Request
                    {
                        RepeatCell = new RepeatCellRequest()
                        {
                            Cell = new CellData
                            {
                                DataValidation = new DataValidationRule()
                                {
                                    Condition = new BooleanCondition
                                    {
                                        Type = "BOOLEAN",

                                    },
                                },
                            },
                            Range = new GridRange()
                            {
                                StartColumnIndex = 0,
                                EndColumnIndex = 1,
                                StartRowIndex = 1,
                                EndRowIndex = cellValues.Count,
                                SheetId = response.sheetId,
                            },
                            Fields = "dataValidation",
                        },
                    },
                })
            };
            SpreadsheetsResource.BatchUpdateRequest updateRequest = this._sheetsService.Spreadsheets.BatchUpdate(request, response.spreadsheet.SpreadsheetId);
            BatchUpdateSpreadsheetResponse? response = updateRequest.Execute();
```

Some notes: 
* To get one column of checkboxes, you need to pass a start and end index that are different by one. Setting them both to 0 in the above example yieds no checkboxes.
* They are unsurprisingly indexed at zero, despite the label of the first column being "1".

So there we go. This was a small piece that made the entire command much more user friendly. With a simple command the user got a ready-to-use sheet shared with them. 
And I made sure they didn't even need to click a few times to add a checkbox column. As always, that last detail took the most time to do out of the entire feature. 


