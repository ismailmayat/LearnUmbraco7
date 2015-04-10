#Examine deep dive#
In this chapter we will cover some of the more advanced topics relating to examine

## Document writing event ##
In chapter we have already covered one of the main bread and butter events GatheringNodeData, you will find that once you start building with examine you will at some point definitely use it.
One of the other useful lower level events, not as commonly used is the DocumentWriting event.  This provides lower level lucene access, with this event you can control how data is injected into the index.

On possible usage is to inject into the index a sortable field.  There are 2 ways of making a field sortable with Examine.  First way is to add sorting attribute to the field in the ExamineIndex.config e.get

<IndexUserFields>            
     <add Name="ArticleDate" EnableSorting="true"/>
</IndexUserFields>

One problem with this approach is that you will need to setup before hand all the fields you want in the index, as it stands the above config would result in only the ArticleDate field being in the index.  In many cases the IndexUserFields
section is empty which means all fields are in the index.  So to be lazy and not put in all the fields then add the sorted field you can use Document writing event and just make the field you want sortable. See code snippet below

```c#

using Examine;
using Umbraco.Core;

namespace MyNamespace
{
    public class ExamineStuff : ApplicationEventHandler
    {
        protected override void ApplicationStarted(UmbracoApplicationBase umbracoApplication, ApplicationContext applicationContext)
        {
            base.ApplicationStarted(umbracoApplication, applicationContext);

            var indexer = (UmbracoContentIndexer)ExamineManager.Instance.IndexProviderCollection[MyIndex];
 
           indexer.DocumentWriting += indexer_DocumentWriting; 
        }

		/// <summary>
       /// used to get low level lucene document access
       /// </summary>
       /// <param name="sender"></param>
       /// <param name="e"></param>
       private void indexer_DocumentWriting(object sender, Examine.LuceneEngine.DocumentWritingEventArgs e)
       {
 
           /*
             for article doc types we need to inject in sortable article date or use update date
             directly into lucene this saves us having to mess around with examineindex.config where you have to put in
             each field then on field u want to sort put in sortable attribute
           */
           if (DocTypeHasArticleDateForSorting(e))
           {
               DateTime articleDate;
               if (e.Fields.ContainsKey(DocumentType.Article.ArticleDate))
               {
                   articleDate = DateTime.Parse(e.Fields[DocumentType.Article.ArticleDate]);
               }
               else
               {
                   articleDate = DateTime.Parse(e.Fields["updateDate"]);
               }
               // in __Sort_articleDate the __ implies is not analysed therefore can be used to sorting. sorted means its retrievable
               //see page 43 lucene in action 2nd edition for full explanation of options
               var field = new Field("__Sort_articleDate", articleDate.ToString("yyyyMMddHHmmss", CultureInfo.InvariantCulture), Field.Store.YES, Field.Index.NOT_ANALYZED);
               
               e.Document.Add(field);
           }
       }
    }
}

```
