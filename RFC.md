*The RFC was made possible with the help of @stevenyomi who originally made one for Tachiyomi*

**Table of Content:**
- [Pagination](#pagination)
- [Extension](#extension)
	- [ExtensionContext](#extensioncontext)
	- [ExtensionItem](#extensionitem)
- [Source](#source)
  - [HttpSource](#httpsource)   
	- [MangaPagingSource](#mangapagingsource)
	- Models
		- Naming
		- Listing
		- Manga
		- Chapter
		- Page/Image
# Pagination
```kt
class PaginatedData<T, K>(
    val data: List<T>,
    val nextKey: K?
) {
    operator fun component1() = data

    operator fun component2() = nextKey
}

infix fun <T, K> List<T>.withNextPageKey(key: K?) = PaginatedData(this, key)
```
# Extension
```kt
interface Extension {  
    fun getItems(context: ExtensionContext): List<ExtensionItem>  
}
```
The entry point. Developers are required to make a class (or object?) with no parameters that inherits `Extension`.  Consumer will use the implemented class to create an `Extension` instance.
## ExtensionContext
`ExtensionContext` is used to pass data to the extension. Like passing the HTTP client and what not.
## ExtensionItem
Sealed class of items that the extension can provides.
# Source
```kt
abstract class Source(context: ExtensionContext): EISource, ExtensionContext by context {

    abstract val id: String

    abstract val name: String

    abstract val language: String

    abstract suspend fun getListings(): List<Listing>
    abstract suspend fun getFilterList(listing: Listing): FilterList
    abstract suspend fun getSearchFilterList(): FilterList

    abstract suspend fun getMangaList(
        listing: Listing,
        page: Int,
        filterState: Map<String, String>,
        filterList: FilterList,
    ): MangaPagingSource

    abstract suspend fun getSearchMangaList(
        query: String,
        page: Int,
        filterState: Map<String, String>,
        filterList: FilterList,
    ): MangaPagingSource

    abstract suspend fun getMangaUpdate(manga: Manga, needDetails: Boolean, needChapters: Boolean): Manga

    abstract suspend fun getPageList(chapter: Chapter): List<Page>

    abstract suspend fun getPage(page: PageUrl): PageComplete // ImageUrl or Text
}
```
## HttpSource
```kt
abstract class HttpSource(context: ExtensionContext): Source(context) {

    val needPcUserAgent: Boolean get() = true

    abstract suspend fun getMangaUrl(manga: Manga): String
    abstract suspend fun getChapterUrl(chapter: Chapter): String

    abstract suspend fun getImageRequest(url: String): HttpRequest

    abstract suspend fun getCoverRequest(manga: Manga): HttpRequest
}
```
## MangaPagingSource
```kt
sealed class MangaPagingSource: PagingSource<MangaPagingSource.Key, Manga>() {
    sealed interface Key
}
```
Implementation: 
```kt
fun <K> MangaPagingSource(
    fetch: suspend (K?) -> PaginatedData<Manga, K>,
): MangaPagingSource = MangaPagingSourceImpl(
    fetch = fetch
)

private class MangaPagingSourceImpl<K>(
    private val fetch: suspend (K?) -> PaginatedData<Manga, K>
): MangaPagingSource() {

    class GenericKey<T>(val value: T): Key

    override suspend fun load(params: LoadParams<Key>): LoadResult<Key, Manga> {
        return try {
            @Suppress("UNCHECKED_CAST")
            val response = fetch((params.key as? GenericKey<K>)?.value)
            LoadResult.Page(
                data = response.data,
                prevKey = null,
                nextKey = response.nextKey?.let(::GenericKey),
            )
        } catch (e: Throwable) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<Key, Manga>): Key? = null
}
```
