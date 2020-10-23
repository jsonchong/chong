### Paging library

The Paging Library makes it easier for you to load data gradually and gracefully(优雅地) within your app's UI.

- A local database that serves as a single source of truth for data presented to the user and manipulated(操控) by the user.
- A web API service.
- A repository that works with the database and the web API service, providing a unified(统一) data interface.
- A ViewModel that provides data specific for the UI.
- The UI, which shows a visual representation of the data in the ViewModel.

The Paging library works with all of these components and coordinates(协调) the interactions between them, so that it can load "pages" of content from a data source and display that content in the UI.

<img src="/Users/zhangchongchong/Library/Application Support/typora-user-images/image-20200610161946974.png" alt="image-20200610161946974" style="zoom:50%;" />

- [**PagedList**](https://developer.android.com/reference/android/arch/paging/PagedList.html) - a collection that loads data in pages, asynchronously. A `PagedList` can be used to load data from sources you define, and present it easily in your UI with a [`RecyclerView`](https://developer.android.com/reference/android/support/v7/widget/RecyclerView.html).
- [**DataSource**](https://developer.android.com/reference/android/arch/paging/DataSource.html) and [**DataSource.Factory**](https://developer.android.com/reference/android/arch/paging/DataSource.Factory.html) - a `DataSource` is the base class for loading snapshots of data(数据快照) into a `PagedList`. A `DataSource.Factory` is responsible for creating a `DataSource`.
- [**LivePagedListBuilder**](https://developer.android.com/reference/android/arch/paging/LivePagedListBuilder.html) - builds a `LiveData<PagedList>`, based on `DataSource.Factory` and a `PagedList.Config`.
- [**BoundaryCallback**](https://developer.android.com/reference/android/arch/paging/PagedList.BoundaryCallback.html) - signals when a `PagedList` has reached the end of available data(当数据到达末尾时)
- [**PagedListAdapter**](https://developer.android.com/reference/android/arch/paging/PagedListAdapter.html) - a `RecyclerView.Adapter` that presents paged data from `PagedLists` in a `RecyclerView`. `PagedListAdapter` listens to `PagedList` loading callbacks as pages are loaded（在页面加载时监听PagedList的回调）, and uses `DiffUtil` to compute fine-grained(细粒度的) updates as new `PagedLists` are received.

### Load data in chunks with the PagedList

In our current implementation, we use a `LiveData<List<Repo>>` to get the data from the database and pass it to the UI. Whenever the data from the local database is modified, the `LiveData` emits an updated list.

The alternative to `List<Repo>` is a `PagedList<Repo>`(替代List<Repo>的方案是PagedList<Repo>). 

A [`PagedList`](https://developer.android.com/reference/android/arch/paging/PagedList.html) is a version of a `List` that loads content in chunks. Similar to the `List`, the `PagedList` holds a snapshot of content, so updates occur when new instances of `PagedList` are delivered(交付) via `LiveData`(PagedList通过LiveData传递).





