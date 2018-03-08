<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<link rel="stylesheet" type="text/css" href="bc.css">
<!--
<script src="run_prettify.js" type="text/javascript"></script>
<script src="https://google-code-prettify.googlecode.com/svn/loader/run_prettify.js" type="text/javascript"></script>
-->
<script src="https://cdn.rawgit.com/google/code-prettify/master/loader/run_prettify.js" type="text/javascript"></script>
</head>

<!---

- open view using UIDocument.ShowElements
  13922857 [Change active document]
  https://forums.autodesk.com/t5/revit-api-forum/change-active-document/m-p/7787792
  https://forums.autodesk.com/t5/revit-api-forum/how-to-open-and-active-a-new-document-that-is-not-saved/m-p/7710749
  Zoom To Awesome!

 #RevitAPI @AutodeskRevit #bim #dynamobim @AutodeskForge #ForgeDevCon

&ndash; 
...

--->

### Switch View or Document by Showing Elements

A recent discussion on using the `ShowElements` method to toggle between documents and views brought up a few interesting points:

- [Open and active an unsaved document](#2) 
- [Zoom to selected elements](#3) 
- [Toggle between documents and views](#4) 

####<a name="2"></a>Open and Active an Unsaved Document

Normally, you can open and switch between documents using the `OpenAndActivateDocument` method.

However, it requires a file path, which may not have been defined yet, in the case of a newly created document.

In that case, you can open and switch between different documents and their views by calling the `UIDocument` `ShowElements` method instead.

This was discussed again in
the [Revit API discussion forum](http://forums.autodesk.com/t5/revit-api-forum/bd-p/160) thread
on [changing the active document](https://forums.autodesk.com/t5/revit-api-forum/change-active-document/m-p/7787792),
and previously
in [how to open and active a new document that is not saved](https://forums.autodesk.com/t5/revit-api-forum/how-to-open-and-active-a-new-document-that-is-not-saved/m-p/7710749):

**Question:** When you have 2 Revit projects open and want to switch between them, it can be done with:

<pre class="code">
application.OpenAndActivateDocument(file).
</pre>

I used `Document.PathName` to get the filename required.

Now I run into problems with files on Revit Server and BIM 360 docs, because their `Document.Pathname` stays empty.

So, my question is, how do I switch between those documents (when `Document.Pathname` is empty)?

Is there a way to switch between documents without having to specify the file name?

**Answer:** The only way I have found is indirectly via `UIDocument.ShowElements`.

You pick an element from the DB document you want to change to, create a `UIDocument` object from a DB document and use `UIDocument.ShowElements`. You have to handle the occasional “No good view found” dialogue, but it seems to always switch the active document regardless of what is found. Helps if it is a `View` specific element.

Not sure if there is a better way...

When you call `UIDocument.ShowElements`, the active `Document` will change to the document you are showing elements in, therefore:

If you filter for any element in one of the views of the newly created document, you can call it using that element to activate that document.

Sometimes you get the dialogue 'No good view found', which is odd, considering you know there is at least one view with the element in considering you are filtering for it by view. You can handle the appearance of this dialogue and the `ActiveDocument` still gets changed. I believe the new document generally has a view with `Elevation` markers, hence there is always something to find and show.

I don't know the implications of changing the `ActiveDocument` this way or why the API has no ability to directly change the `ActiveDocument` directly, but I suspect if it were easy in terms of how the API works, it would have been done by now.

This workaround was also mentioned in The Building Coder discussion
on [mirroring in a new family and changing active view](http://thebuildingcoder.typepad.com/blog/2010/11/mirroring-in-a-new-family-and-changing-active-view.html):

> The first issue that arises is that the mirror command requires a current active view, which is not automatically present in the family document. Joe discovers a workaround for that issue using the `ShowElements` method. It generates an unwanted warning message, so a second step is required to deal with eliminating that as well.

> As you can see on reading the final solution carefully, you can use the `ShowElements` method to change the active view and even switch it between the family and project documents. The official Revit 2011 API does not provide any method to switch the active view, but using `ShowElements` can be used to create a workaround for that."

I implemented a new external
command [CmdSwitchDoc](https://github.com/jeremytammik/the_building_coder_samples/blob/master/BuildingCoder/BuildingCoder/CmdSwitchDoc.cs)
in [The Building Coder samples](https://github.com/jeremytammik/the_building_coder_samples) to try this out, in
[release 2018.0.138.0](https://github.com/jeremytammik/the_building_coder_samples/releases/tag/2018.0.138.0).

It demonstrates two uses of the the `ShowElements` method:

- [Zoom to selected elements](#3)
- [Toggle between documents and views](#4)

####<a name="3"></a>Zoom to Selected Elements

<pre class="code">
<span style="color:gray;">///</span><span style="color:green;">&nbsp;</span><span style="color:gray;">&lt;</span><span style="color:gray;">summary</span><span style="color:gray;">&gt;</span>
<span style="color:gray;">///</span><span style="color:green;">&nbsp;Zoom&nbsp;to&nbsp;the&nbsp;given&nbsp;elements,&nbsp;switching&nbsp;view&nbsp;if&nbsp;needed.</span>
<span style="color:gray;">///</span><span style="color:green;">&nbsp;</span><span style="color:gray;">&lt;/</span><span style="color:gray;">summary</span><span style="color:gray;">&gt;</span>
<span style="color:gray;">///</span><span style="color:green;">&nbsp;</span><span style="color:gray;">&lt;</span><span style="color:gray;">param</span><span style="color:gray;">&nbsp;name</span><span style="color:gray;">=</span><span style="color:gray;">&quot;</span>ids<span style="color:gray;">&quot;</span><span style="color:gray;">&gt;&lt;/</span><span style="color:gray;">param</span><span style="color:gray;">&gt;</span>
<span style="color:gray;">///</span><span style="color:green;">&nbsp;</span><span style="color:gray;">&lt;</span><span style="color:gray;">param</span><span style="color:gray;">&nbsp;name</span><span style="color:gray;">=</span><span style="color:gray;">&quot;</span>message<span style="color:gray;">&quot;</span><span style="color:gray;">&gt;</span><span style="color:green;">Error&nbsp;message&nbsp;on&nbsp;failure</span><span style="color:gray;">&lt;/</span><span style="color:gray;">param</span><span style="color:gray;">&gt;</span>
<span style="color:gray;">///</span><span style="color:green;">&nbsp;</span><span style="color:gray;">&lt;</span><span style="color:gray;">param</span><span style="color:gray;">&nbsp;name</span><span style="color:gray;">=</span><span style="color:gray;">&quot;</span>elements<span style="color:gray;">&quot;</span><span style="color:gray;">&gt;</span><span style="color:green;">Elements&nbsp;causing&nbsp;failure</span><span style="color:gray;">&lt;/</span><span style="color:gray;">param</span><span style="color:gray;">&gt;</span>
<span style="color:gray;">///</span><span style="color:green;">&nbsp;</span><span style="color:gray;">&lt;</span><span style="color:gray;">returns</span><span style="color:gray;">&gt;&lt;/</span><span style="color:gray;">returns</span><span style="color:gray;">&gt;</span>
<span style="color:#2b91af;">Result</span>&nbsp;ZoomToElements(&nbsp;
&nbsp;&nbsp;<span style="color:#2b91af;">UIDocument</span>&nbsp;uidoc,
&nbsp;&nbsp;<span style="color:#2b91af;">ICollection</span>&lt;<span style="color:#2b91af;">ElementId</span>&gt;&nbsp;ids,
&nbsp;&nbsp;<span style="color:blue;">ref</span>&nbsp;<span style="color:blue;">string</span>&nbsp;message,
&nbsp;&nbsp;<span style="color:#2b91af;">ElementSet</span>&nbsp;elements&nbsp;)
{
&nbsp;&nbsp;<span style="color:blue;">int</span>&nbsp;n&nbsp;=&nbsp;ids.Count;
 
&nbsp;&nbsp;<span style="color:blue;">if</span>(&nbsp;0&nbsp;==&nbsp;n&nbsp;)
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;message&nbsp;=&nbsp;<span style="color:#a31515;">&quot;Please&nbsp;select&nbsp;at&nbsp;least&nbsp;one&nbsp;element&nbsp;to&nbsp;zoom&nbsp;to.&quot;</span>;
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:blue;">return</span>&nbsp;<span style="color:#2b91af;">Result</span>.Cancelled;
&nbsp;&nbsp;}
&nbsp;&nbsp;<span style="color:blue;">try</span>
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;uidoc.ShowElements(&nbsp;ids&nbsp;);
&nbsp;&nbsp;}
&nbsp;&nbsp;<span style="color:blue;">catch</span>
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#2b91af;">Document</span>&nbsp;doc&nbsp;=&nbsp;uidoc.Document;
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:blue;">foreach</span>(&nbsp;<span style="color:#2b91af;">ElementId</span>&nbsp;id&nbsp;<span style="color:blue;">in</span>&nbsp;ids&nbsp;)
&nbsp;&nbsp;&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#2b91af;">Element</span>&nbsp;e&nbsp;=&nbsp;doc.GetElement(&nbsp;id&nbsp;);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;elements.Insert(&nbsp;e&nbsp;);
&nbsp;&nbsp;&nbsp;&nbsp;}
 
&nbsp;&nbsp;&nbsp;&nbsp;message&nbsp;=&nbsp;<span style="color:blue;">string</span>.Format(&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#a31515;">&quot;Cannot&nbsp;zoom&nbsp;to&nbsp;element{0}.&quot;</span>,
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1&nbsp;==&nbsp;n&nbsp;?&nbsp;<span style="color:#a31515;">&quot;&quot;</span>&nbsp;:&nbsp;<span style="color:#a31515;">&quot;s&quot;</span>&nbsp;);
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:blue;">return</span>&nbsp;<span style="color:#2b91af;">Result</span>.Failed;
&nbsp;&nbsp;}
&nbsp;&nbsp;<span style="color:blue;">return</span>&nbsp;<span style="color:#2b91af;">Result</span>.Succeeded;
}
</pre>

This functionality is similar to that provided by
the [Zoom to Awesome! add-in](https://bimopedia.com/2013/04/02/zoom-to-awesome) by Phil Read.

Here is a 40-second demo by Luke Johnson 
of [using Zoom to Awesome](https://knowledge.autodesk.com/support/revit-products/getting-started/caas/screencast/Main/Details/8e9a043d-9383-496b-8e86-6ec3ab055c0e.html),
also showing how to add a keyboard shortcut:

<center>
<iframe width="400" height="470" src="https://screencast.autodesk.com/Embed/Timeline/8e9a043d-9383-496b-8e86-6ec3ab055c0e" frameborder="0" allowfullscreen webkitallowfullscreen></iframe>
</center>

####<a name="4"></a>Toggle Between Documents and Views

<pre class="code">
<span style="color:gray;">///</span><span style="color:green;">&nbsp;</span><span style="color:gray;">&lt;</span><span style="color:gray;">summary</span><span style="color:gray;">&gt;</span>
<span style="color:gray;">///</span><span style="color:green;">&nbsp;Toggle&nbsp;back&nbsp;and&nbsp;forth&nbsp;between&nbsp;two&nbsp;different&nbsp;documents</span>
<span style="color:gray;">///</span><span style="color:green;">&nbsp;</span><span style="color:gray;">&lt;/</span><span style="color:gray;">summary</span><span style="color:gray;">&gt;</span>
<span style="color:blue;">void</span>&nbsp;ToggleViews(&nbsp;
&nbsp;&nbsp;<span style="color:#2b91af;">View</span>&nbsp;view1,&nbsp;
&nbsp;&nbsp;<span style="color:blue;">string</span>&nbsp;filepath2&nbsp;)
{
&nbsp;&nbsp;<span style="color:#2b91af;">Document</span>&nbsp;doc&nbsp;=&nbsp;view1.Document;
&nbsp;&nbsp;<span style="color:#2b91af;">UIDocument</span>&nbsp;uidoc&nbsp;=&nbsp;<span style="color:blue;">new</span>&nbsp;<span style="color:#2b91af;">UIDocument</span>(&nbsp;doc&nbsp;);
&nbsp;&nbsp;<span style="color:#2b91af;">Application</span>&nbsp;app&nbsp;=&nbsp;doc.Application;
&nbsp;&nbsp;<span style="color:#2b91af;">UIApplication</span>&nbsp;uiapp&nbsp;=&nbsp;<span style="color:blue;">new</span>&nbsp;<span style="color:#2b91af;">UIApplication</span>(&nbsp;app&nbsp;);
 
&nbsp;&nbsp;<span style="color:green;">//&nbsp;Select&nbsp;some&nbsp;elements&nbsp;in&nbsp;the&nbsp;first&nbsp;document</span>
 
&nbsp;&nbsp;<span style="color:#2b91af;">ICollection</span>&lt;<span style="color:#2b91af;">ElementId</span>&gt;&nbsp;idsView1
&nbsp;&nbsp;&nbsp;&nbsp;=&nbsp;<span style="color:blue;">new</span>&nbsp;<span style="color:#2b91af;">FilteredElementCollector</span>(&nbsp;doc,&nbsp;view1.Id&nbsp;)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.WhereElementIsNotElementType()
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.ToElementIds();
 
&nbsp;&nbsp;<span style="color:green;">//&nbsp;Open&nbsp;the&nbsp;second&nbsp;file</span>
 
&nbsp;&nbsp;<span style="color:#2b91af;">UIDocument</span>&nbsp;uidoc2&nbsp;=&nbsp;uiapp
&nbsp;&nbsp;&nbsp;&nbsp;.OpenAndActivateDocument(&nbsp;filepath2&nbsp;);
 
&nbsp;&nbsp;<span style="color:#2b91af;">Document</span>&nbsp;doc2&nbsp;=&nbsp;uidoc2.Document;
 
&nbsp;&nbsp;<span style="color:green;">//&nbsp;Do&nbsp;something&nbsp;in&nbsp;second&nbsp;file</span>
 
&nbsp;&nbsp;<span style="color:blue;">using</span>(&nbsp;<span style="color:#2b91af;">Transaction</span>&nbsp;tx&nbsp;=&nbsp;<span style="color:blue;">new</span>&nbsp;<span style="color:#2b91af;">Transaction</span>(&nbsp;doc2&nbsp;)&nbsp;)
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;tx.Start(&nbsp;<span style="color:#a31515;">&quot;Change&nbsp;Scale&quot;</span>&nbsp;);
&nbsp;&nbsp;&nbsp;&nbsp;doc2.ActiveView.get_Parameter(
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:#2b91af;">BuiltInParameter</span>.VIEW_SCALE_PULLDOWN_METRIC&nbsp;)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;.Set(&nbsp;20&nbsp;);
&nbsp;&nbsp;&nbsp;&nbsp;tx.Commit();
&nbsp;&nbsp;}
 
&nbsp;&nbsp;<span style="color:green;">//&nbsp;Save&nbsp;modified&nbsp;second&nbsp;file</span>
 
&nbsp;&nbsp;<span style="color:#2b91af;">SaveAsOptions</span>&nbsp;opt&nbsp;=&nbsp;<span style="color:blue;">new</span>&nbsp;<span style="color:#2b91af;">SaveAsOptions</span>
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;OverwriteExistingFile&nbsp;=&nbsp;<span style="color:blue;">true</span>
&nbsp;&nbsp;};
 
&nbsp;&nbsp;doc2.SaveAs(&nbsp;filepath2,&nbsp;opt&nbsp;);
 
&nbsp;&nbsp;<span style="color:green;">//&nbsp;Switch&nbsp;back&nbsp;to&nbsp;original&nbsp;file;</span>
&nbsp;&nbsp;<span style="color:green;">//&nbsp;in&nbsp;a&nbsp;new&nbsp;file,&nbsp;doc.PathName&nbsp;is&nbsp;empty</span>
 
&nbsp;&nbsp;<span style="color:blue;">if</span>(&nbsp;!<span style="color:blue;">string</span>.IsNullOrEmpty(&nbsp;doc.PathName&nbsp;)&nbsp;)
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;uiapp.OpenAndActivateDocument(
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;doc.PathName&nbsp;);
 
&nbsp;&nbsp;&nbsp;&nbsp;doc2.Close(&nbsp;<span style="color:blue;">false</span>&nbsp;);&nbsp;<span style="color:green;">//&nbsp;no&nbsp;problem&nbsp;here,&nbsp;says&nbsp;Remy</span>
&nbsp;&nbsp;}
&nbsp;&nbsp;<span style="color:blue;">else</span>
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:green;">//&nbsp;Avoid&nbsp;using&nbsp;OpenAndActivateDocument</span>
 
&nbsp;&nbsp;&nbsp;&nbsp;uidoc.ShowElements(&nbsp;idsView1&nbsp;);
&nbsp;&nbsp;&nbsp;&nbsp;uidoc.RefreshActiveView();
 
&nbsp;&nbsp;&nbsp;&nbsp;<span style="color:green;">//doc2.Close(&nbsp;false&nbsp;);&nbsp;//&nbsp;Remy&nbsp;says:&nbsp;Revit&nbsp;throws&nbsp;the&nbsp;exception&nbsp;and&nbsp;doesn&#39;t&nbsp;close&nbsp;the&nbsp;file</span>
&nbsp;&nbsp;}
}
</pre>

<center>
<img src="img/toggle.png" alt="Toggle" width="183"/>
</center>