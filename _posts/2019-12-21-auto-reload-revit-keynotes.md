---
layout: post
title: Auto Reload Revit Keynotes
author: Zachary Mathews
tags: revit 2019 addin keynotes multithreading file-system IExternalApplication
---

As of Revit 2019, users are still required to manually reload keynote tables and edit them in an external program. This post tackles the first problem by developing a custom Revit addin that watches the file system and automatically reloads the keynote table when a change is detected. Instances where the user loads a new keynote file are handled appropriately and the addin supports multiple open projects.

## The Interface
The addin implements the IExternalApplication interface to allow behind the scenes tracking of Revit keynote files. To fully implement the IExternalApplication interface, the addin must define the two methods:
- `Result OnStartup(UIControlledApplication application)`
- `Result OnShutdown(UIControlledApplication application)`

The addin defines the two methods as such:
```c#
Result OnStartup(UIControlledApplication uiapp)
{
  // Register event handlers
  uiapp.ControlledApplication.DocumentOpened += new EventHandler<DocumentOpenedEventArgs>(OnDocumentOpen);
  uiapp.ControlledApplication.DocumentClosing += new EventHandler<DocumentClosingEventArgs>(OnDocumentClosing);

  return Result.Succeeded;
}

Result OnShutdown(UIControlledApplication uiapp)
{
  // Unregister event handlers
  uiapp.ControlledApplication.DocumentOpened -= OnDocumentOpen;
  uiapp.ControlledApplication.DocumentClosing -= OnDocumentClosing;

  return Result.Succeeded;
}
```
In the first method we're assigning new handlers to the `DocumentOpened` and `DocumentClosing` events created by Revit. We will define the event handler methods `OnDocumentOpen` and `OnDocumentClosing` later on. In the second method we're removing the handlers from the `DocumentOpened` and `DocumentClosing` event so that Revit will not try to call them after our addin exits.

## Watching the Keynote Files
In order to keep track of the open projects a dictionary of TrackedDocuments along with their hashes is used:
- `private static Dictionary<int, TrackedDocument> trackedDocuments = new Dictionary<int, TrackedDocument>();`

The TrackedDocument class is defined as such.
```c#
public class TrackedDocument
{
    private Document document;
    public int hash;
    private KeynoteTable keynoteTable;
    public string keynotePath;
    public bool keynoteFileChanged;

    public TrackedDocument(Document document)
    {
        this.document = document;
        hash = document.GetHashCode();
        keynoteTable = KeynoteTable.GetKeynoteTable(this.document);
        keynotePath = ModelPathUtils.ConvertModelPathToUserVisiblePath(keynoteTable.GetExternalFileReference().GetAbsolutePath());
        keynotesUpdated = false;
    }

    public void UpdateKeynoteTable()
    {
        using (Transaction transaction = new Transaction(document))
        {
            if (TransactionStatus.Started == transaction.Start("Update keynote tables"))
            {
                KeyBasedTreeEntriesLoadResults reloadResults = new KeyBasedTreeEntriesLoadResults();
                ExternalResourceLoadStatus reloadStatus = keynoteTable.Reload(reloadResults);

                if (ExternalResourceLoadStatus.Success == reloadStatus || ExternalResourceLoadStatus.ResourceAlreadyCurrent == reloadStatus)
                {
                    if (TransactionStatus.Committed == transaction.Commit())
                    {
                        keynoteFileChanged = false;
                    }
                }
            }
        }
    }
    public void KeynotePathChanged()
    {
        keynotePath = ModelPathUtils.ConvertModelPathToUserVisiblePath(keynoteTable.GetExternalFileReference().GetAbsolutePath());
        keynotesUpdated = false;
    }
}
```
Each TrackedDocument object keeps track of its own document, keynote file,


## Reloading the Keynote Table
In order to reload the flagged keynote tables, another event handler must be registered. The Revit API makes the `Idling` event available which the add-in uses to reload the keynotes table between user operations. The `Idling` event handler is registered and unregistered in the `OnStartup` and `OnShutdown` methods respectively.
```c#
Result OnStartup(UIControlledApplication uiapp)
{
  // Register event handlers
  uiapp.ControlledApplication.DocumentOpened += new EventHandler<DocumentOpenedEventArgs>(OnDocumentOpen);
  uiapp.ControlledApplication.DocumentClosing += new EventHandler<DocumentClosingEventArgs>(OnDocumentClosing);
  uiapp.Idling += new EventHandler<Autodesk.Revit.UI.Events.IdlingEventArgs>(OnIdling);

  return Result.Succeeded;
}

Result OnShutdown(UIControlledApplication uiapp)
{
  // Unregister event handlers
  uiapp.ControlledApplication.DocumentOpened -= OnDocumentOpen;
  uiapp.ControlledApplication.DocumentClosing -= OnDocumentClosing;
  uiapp.Idling -= OnIdling;

  return Result.Succeeded;
}
```

```c#
public void OnIdling(object sender, IdlingEventArgs e)
{
    foreach (KeyValuePair<int, TrackedDocument> kvp in trackedDocuments)
    {
        int hash = kvp.Key;
        TrackedDocument trackedDocument = kvp.Value;

        if (trackedDocument.keynoteFileChanged == true)
        {
            trackedDocument.UpdateKeynoteTable();
        }
    }
}
```

```c#
public void UpdateKeynoteTable()
{
    using (Transaction transaction = new Transaction(document))
    {
        if (TransactionStatus.Started == transaction.Start("Update keynote tables"))
        {
            KeyBasedTreeEntriesLoadResults reloadResults = new KeyBasedTreeEntriesLoadResults();
            ExternalResourceLoadStatus reloadStatus = keynoteTable.Reload(reloadResults);

            if (ExternalResourceLoadStatus.Success == reloadStatus || ExternalResourceLoadStatus.ResourceAlreadyCurrent == reloadStatus)
            {
                if (TransactionStatus.Committed == transaction.Commit())
                {
                    keynotesUpdated = false;
                }
            }
        }
    }
}
```

## Updating the Keynote File Location
```c#
Result OnStartup(UIControlledApplication uiapp)
{
  // Register event handlers
  uiapp.ControlledApplication.DocumentOpened += new EventHandler<DocumentOpenedEventArgs>(OnDocumentOpen);
  uiapp.ControlledApplication.DocumentClosing += new EventHandler<DocumentClosingEventArgs>(OnDocumentClosing);
  uiapp.Idling += new EventHandler<Autodesk.Revit.UI.Events.IdlingEventArgs>(OnIdling);
  uiapp.ControlledApplication.DocumentChanged += new EventHandler<DocumentChangedEventArgs>(OnDocumentChanged);

  return Result.Succeeded;
}

Result OnShutdown(UIControlledApplication uiapp)
{
  // Unregister event handlers
  uiapp.ControlledApplication.DocumentOpened -= OnDocumentOpen;
  uiapp.ControlledApplication.DocumentClosing -= OnDocumentClosing;
  uiapp.Idling -= OnIdling;
  uiapp.DocumentChanged -= OnDocumentChanged;

  return Result.Succeeded;
}
```
