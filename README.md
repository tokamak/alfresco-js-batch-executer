Alfresco JavaScript Batch Executer
==================================

Have you ever found yourself updating thousands of nodes in Alfresco using a JavaScript script?
For example to post-process documents imported by BFSIT? Or did you ever have to traverse a large folder tree and
run some logic on each document? Then you know that repo-side scripts run slow and in one transaction:
you may wait a day for it to finish, and if it fails then nothing is saved. It makes it almost impossible to
use repo-side JavaScript for bulk processing.

Alfresco JavaScript Batch Executer tool is aimed to solve this problem in **multithreaded** and
**transactional** manner. It can do the job 10 times faster, while clearly showing the progress and also
allowing you to cancel running jobs.

Usage
-----

Once you install the tool you will have a new root object available in your scripts: `batchExecuter`.
Here is an example usage which will set an author for all documents in Alfresco:

```javascript
batchExecuter.processFolderRecursively({
    root: companyhome,
    onNode: function(node) {
        if (node.isDocument) {
            node.properties['cm:author'] = "Ciber NL";
            node.save();
        }
    }
});
```

This simple script will traverse all folders and documents in Alfresco recursively and set cm:author value to
"Ciber NL" for each document. 4 threads will be processing nodes, and each batch of 200 documents will be
committed in a separate transaction.

Here is another example which lets you process a CSV file in a highly-performing way:

```javascript
batchExecuter.processArray({
    items: companyhome.childByNamePath("groups.csv").content.split("\n"),
    batchSize: 50,
    threads: 2,
    onNode: function(row) {
        // split row string into columns and create a group with given name, for example
    }
});
```

Here is a last example which lets you reset the permission inheritance for a whole repository (took 2 days to reset the permissions of 4 Million nodes) :

```javascript
function inheritanceReset(parentNode) {
	logger.log("Processing " + parentNode.name + "...");
	batchExecuter.processFolderRecursively({
		root: parentNode,
		batchSize: 10000,
		threads: 1,
			onNode: function(node){
				if(node.inheritsPermissions()){
				  node.setInheritsPermissions(false);
				  node.save();
				  node.setInheritsPermissions(true);
				  node.save();
				}
			}
		});
   }
logger.log("START - Permission Inheritance RESET");
inheritanceReset(search.selectNodes("/sys:system/sys:people/.")[0]);
inheritanceReset(search.selectNodes("/app:company_home/app:guest_home/.")[0]);
inheritanceReset(search.selectNodes("/app:company_home/app:user_homes/.")[0]);
//inheritanceReset(search.selectNodes("/app:company_home/st:sites/.")[0]);
logger.log("END - Permission Inheritance RESET");
```

You can monitor the progress in log files and control running jobs using a webscript page:

http://localhost:8080/alfresco/s/ciber/batch-executer/jobs

![alt text](/screenshot.png "Jobs page screenshot")

Parameters
----------

Batch executer supports following functions to process bulks of data:

* `processFolderRecursively(parametersObject)` - processes a folder recursively. Parameter `root` specifies where to start.
* `processArray(parametersObject)` - processes an array of items: it may be nodes or primitive JavaScript objects or anything.
Parameter `items` contains the array.

Following parameters are supported when calling these functions.

<table>
<thead>
<tr>
    <th>Name</th>
    <th>Description</th>
</tr>
</thead>
<tbody>
<tr>
    <td><code>root</code></td>
    <td>
        The folder to process, mandatory when calling <code>processFolderRecursively</code> function, ignored otherwise.
        The folder is traversed in depth-first-search manner and <strong>all</strong> nodes are fed to
        the processing function, including the root folder itself and any sub-folders and documents.
        Only <code>cm:contains</code> associations are used to fetch children unless <b>browseSystemNodes</b> parameter is set to true 			then <code>sys:children</code> (for <code>sys:container</code>) , <code>rn:rendition</code> (for <code>cm:content</code>) , <code>cm:failedThumbnail</code> (for <code>cm:content</code>) will be fetched as well.
       	WARNING : <code> cm:category_root</code>,<code>cm:mlRoot</code>,<code>cm:mlRoot</code>,<code>cm:mlRoot</code> are not yet supported
    </td>
</tr>
<tr>
    <td><code>items</code></td>
    <td>
        The array of items to process, mandatory when calling <code>processArray</code> function, ignored otherwise.
        Each item is fed to processing function <code>onNode</code> or <code>onBatch</code> and does not necessarily
        have to be a node. It may be any JavaScript object.
    </td>
</tr>
<tr>
    <td><code>batchSize</code></td>
    <td>
        The size of a batch to use when processing. Optional, default value is <code>200</code>.
        Each batch is committed in separate transaction.
    </td>
</tr>
<tr>
    <td><code>threads</code></td>
    <td>
        The number of processing threads. Optional, default value is <code>4</code>.
    </td>
</tr>
<tr>
    <td><code>disableRules</code></td>
    <td>
        May be used to disable Alfresco rules when processing takes place. Optional, <code>false</code> by default.
    </td>
</tr>
<tr>
    <td><code>onNode</code></td>
    <td>
        A JavaScript function which will be executed on each item found by <code>batchExecuter</code>. It receives one
        parameter: the item, it may be a document, folder, a string from <code>items</code> array etc. Mandatory unless
        <code>onBatch</code> function is supplied.
    </td>
</tr>
<tr>
    <td><code>onBatch</code></td>
    <td>
        A JavaScript function which will be executed on each <strong>batch</strong> of items to process.
        It receives one parameter: a JavaScript array of items in the batch. This function can be used to further
        improve performance by grouping some logic in batches. For example if you have to check for each document if
        another one exists with the same name, then you can make <strong>one</strong> query with all names included by
        <code>OR</code> instead of executing one search query for each node. This can improve performance but
        complicates the implementation of course. <code>onBatch</code> parameter is mandatory unless
        <code>onNode</code> function is present.
    </td>
</tr>
</tbody>
</table>
