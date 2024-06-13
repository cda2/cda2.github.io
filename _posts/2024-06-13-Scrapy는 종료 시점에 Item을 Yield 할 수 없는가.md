---
layout: post
title:  "Scrapy는 종료 시점에 Item 을 Yield 할 수 없는가"
date:   2024-06-13 01:10:51 +0900
categories: [Scrapy]
---

## 개요
`Scrapy` 프레임워크는 특성 상 종료 시점에 `Item` 을 `Yield` 할 수 없는데, 간단한 우회법으로 이를 가능하게 해본다.

## TL;DR
최소 두 가지 기능을 사용하면 **상당히 지저분하지만** 아키텍처나 Scrapy의 기본 구조를 뒤집어 엎지 않고도 종료 시점에 `Item`을 `Yield` 처리 할 수 있다.
1. `Scrapy` 프레임워크 내부에 있는 [Scraper](https://github.com/scrapy/scrapy/blob/1282ddf8f77299edf613679c2ee0b606e96808ce/scrapy/core/scraper.py#L110) 를 사용할 것
2. `Twisted` 의 [Deferred](https://docs.twisted.org/en/stable/core/howto/defer.html) 객체를 사용하여 Signal 기반으로 처리할 것

여러가지 방법이 있지만, 위의 절차를 가장 쉽고 빠르게 적용할 수 있는 방법은 다음과 같다.
1. Scraper 클래스의 [`handle_spider_output`](https://github.com/scrapy/scrapy/blob/1282ddf8f77299edf613679c2ee0b606e96808ce/scrapy/core/scraper.py#L272-L308) 메서드를 적용한다.
2. 적용한 결과는 `Deferred` 객체이므로 이를 시그널 처리가 가능한 메서드 단에서 반환한다.
- `close_spider` 메서드는 `Deferred` 기반의 시그널 처리를 지원한다.
- `from_crawler` 등에서 시그널 연결을 기본적으로 지원하므로, 구현하기 편리하다.

간단한 단 건 처리 예시를 작성해보면 다음과 같다.
```python
	...
	def close_spider(self, spider) -> Deferred
		...
		item: Item = Item(...)
		items = [item]
		fake_req: Request = Request(...)
		fake_res: Response = Response(...)
		item_dfd: Deferred = spider.crawler.engine.scraper.handle_spider_output(items, fake_req, fake_res, spider)
		return item_dfd
```

여러 건을 한 번에 처리해야 하는 케이스를 작성해보면 다음과 같다.
```python
	...
	def close_spider(self, spider) -> Deferred
		...
		items: list[Item] = [...]
		fake_req: Request = Request(...)
		fake_res: Response = Response(...)
		parallel_item_dfd: Deferred = spider.crawler.engine.scraper.handle_spider_output(items, fake_req, fake_res, spider)
		return parallel_item_dfd

```

여러 건을 나누어서 (페이징 등) 처리해야 하는 케이스를 작성해보면 다음과 같다.
- 원격 DB 를 사용하고 있어, 데이터베이스에 부하를 줄이려는 목적 등의 경우 사용

```python
from twisted.internet.defer import gatherResults
	...
	def close_spider(self, spider) -> Deferred
		...
		dfd_list: list[Deferred] = []
		batch_size: int = ...
		fake_req: Request = Request(...)
		fake_res: Response = Response(...)

		for batch in ...:
			items: list[Item] = [...]
			part_parallel_dfd: Deferred = spider.crawler.engine.scraper.handle_spider_output(items, fake_req, fake_res, spider)
			dfd_list.append(part_parallel_dfd)
			
		return gatherResults(dfd_list)
```

## 상세 설명

`handle_spider_output` 메서드는 다음과 같이 정의되어 있다.

```python 
    def handle_spider_output(
        self,
        result: Union[Iterable, AsyncIterable],
        request: Request,
        response: Union[Response, Failure],
        spider: Spider,
    ) -> Deferred:
        if not result:
            return defer_succeed(None)
        # errback 이 적용된 result 래핑 객체
        it: Union[Generator, AsyncGenerator]
        # spider에서 Request | Item 을 yield 한 메서드가 비동기 메서드인 경우, 비동기 병렬 처리
        if isinstance(result, AsyncIterable):
            it = aiter_errback(
                result, self.handle_spider_error, request, response, spider
            )
            dfd = parallel_async(
                it,
                self.concurrent_items,
                self._process_spidermw_output,
                request,
                response,
                spider,
            )
        # 메서드가 동기 메서드인 경우, 병렬 처리
        else:
            it = iter_errback(
                result, self.handle_spider_error, request, response, spider
            )
            dfd = parallel(
                it,
                self.concurrent_items,
                self._process_spidermw_output,
                request,
                response,
                spider,
            )
        return dfd
```

내부에서 사용하고 있는 `_process_spidermw_output` 메서드는 다음과 같다.

```python
    def _process_spidermw_output(
        self, output: Any, request: Request, response: Response, spider: Spider
    ) -> Optional[Deferred[Any]]:
        """Process each Request/Item (given in the output parameter) returned
        from the given spider
        """
        assert self.slot is not None  # typing
        # 요청 객체인 경우 engine 에 요청을 스케줄링한다.
        if isinstance(output, Request):
            assert self.crawler.engine is not None  # typing
            self.crawler.engine.crawl(request=output)
        # Item 객체인 경우 `itemproc` 에게 처리를 위임한다.
        elif is_item(output):
            self.slot.itemproc_size += 1
            dfd = self.itemproc.process_item(output, spider)
            dfd.addBoth(self._itemproc_finished, output, response, spider)
            return dfd
        elif output is None:
            pass
        else:
            typename = type(output).__name__
            logger.error(
                "Spider must return request, item, or None, got %(typename)r in %(request)s",
                {"request": request, "typename": typename},
                extra={"spider": spider},
            )
        return None

```

두 메서드를 좀 살펴보면, `itemproc` 변수에게 많은 것을 위임하고 있는 것을 알 수 있다. `itemproc` 은 `Item Processor` 의 약자이다.
이에 대해서 너무 깊이 들여다 볼 필요는 없지만, 적어도 `itemproc`이 무슨 역할을 수행하는지 정도는 알아야 한다.
### Item Processor

다시 `Scraper` 클래스로 돌아가보자. 생성자 메서드를 보면 다음과 같이 정의되어 있다.

```python
class Scraper:
    def __init__(self, crawler: Crawler) -> None:
        self.slot: Optional[Slot] = None
        self.spidermw: SpiderMiddlewareManager = SpiderMiddlewareManager.from_crawler(
            crawler
        )
        itemproc_cls: Type[ItemPipelineManager] = load_object(
            crawler.settings["ITEM_PROCESSOR"]
        )
        self.itemproc: ItemPipelineManager = itemproc_cls.from_crawler(crawler)
        self.concurrent_items: int = crawler.settings.getint("CONCURRENT_ITEMS")
        self.crawler: Crawler = crawler
        self.signals: SignalManager = crawler.signals
        assert crawler.logformatter
        self.logformatter: LogFormatter = crawler.logformatter
```

살펴보면 `itemproc` 필드는 다음과 같이 생성됨을 알 수 있다.
1. `Settings` 객체로부터 `ITEM_PROCESSOR` 설정 값을 추출
2. 추출한 클래스 객체를 `from_crawler` 메서드를 호출하여 인스턴스로 만든 후, 인스턴스 변수에 저장

`Scrapy` 프레임워크의 문서 등을 살펴봐도 `ITEM_PROCESSOR` 에 대한 설명은 자세히 나와있지 않다.  
이는 내부 구현 클래스이고, 프레임워크의 아키텍처를 뒤집어 엎을 생각이 아닌 이상 그럴 필요가 없기 때문이기도 하다.

실제로 내부적으로 쓰이는 기본 설정 값은 [default_settings.py](https://github.com/scrapy/scrapy/blob/master/scrapy/settings/default_settings.py#L205) 파일에서 찾을 수 있는데, 기본값으로 `scrapy.pipelines.ItemPipelineManager` 를 사용하고 있다.  
이 클래스를 scrapy 내부에서 찾아보면 다음과 같은 클래스가 발견된다.

```python
"""  
Item pipeline  
  
See documentation in docs/item-pipeline.rst  
"""  
from typing import Any, List  
  
from twisted.internet.defer import Deferred  
  
from scrapy import Spider  
from scrapy.middleware import MiddlewareManager  
from scrapy.utils.conf import build_component_list  
from scrapy.utils.defer import deferred_f_from_coro_f  
  
  
class ItemPipelineManager(MiddlewareManager):  
    component_name = "item pipeline"  
  
    @classmethod  
    def _get_mwlist_from_settings(cls, settings) -> List[Any]:  
        return build_component_list(settings.getwithbase("ITEM_PIPELINES"))  
  
    def _add_middleware(self, pipe: Any) -> None:  
        super()._add_middleware(pipe)  
        if hasattr(pipe, "process_item"):  
            self.methods["process_item"].append(  
                deferred_f_from_coro_f(pipe.process_item)  
            )  
    def process_item(self, item: Any, spider: Spider) -> Deferred:  
        return self._process_chain("process_item", item, spider)
```

`ItemPipelineManager` 클래스가 상속하고 있는 `MiddlewareManager` 클래스의 코드는 다음과 같다.

```python
from __future__ import annotations

import logging
import pprint
from collections import defaultdict, deque
from typing import (
    TYPE_CHECKING,
    Any,
    Callable,
    Deque,
    Dict,
    Iterable,
    List,
    Optional,
    Tuple,
    Union,
    cast,
)

from twisted.internet.defer import Deferred

from scrapy import Spider
from scrapy.exceptions import NotConfigured
from scrapy.settings import Settings
from scrapy.utils.defer import process_chain, process_parallel
from scrapy.utils.misc import create_instance, load_object

if TYPE_CHECKING:
    # typing.Self requires Python 3.11
    from typing_extensions import Self

    from scrapy.crawler import Crawler


logger = logging.getLogger(__name__)


class MiddlewareManager:
    """Base class for implementing middleware managers"""

    component_name = "foo middleware"

    def __init__(self, *middlewares: Any) -> None:
        self.middlewares = middlewares
        # Only process_spider_output and process_spider_exception can be None.
        # Only process_spider_output can be a tuple, and only until _async compatibility methods are removed.
        self.methods: Dict[
            str, Deque[Union[None, Callable, Tuple[Callable, Callable]]]
        ] = defaultdict(deque)
        for mw in middlewares:
            self._add_middleware(mw)

    @classmethod
    def _get_mwlist_from_settings(cls, settings: Settings) -> List[Any]:
        raise NotImplementedError

    @classmethod
    def from_settings(
        cls, settings: Settings, crawler: Optional[Crawler] = None
    ) -> Self:
        mwlist = cls._get_mwlist_from_settings(settings)
        middlewares = []
        enabled = []
        for clspath in mwlist:
            try:
                mwcls = load_object(clspath)
                mw = create_instance(mwcls, settings, crawler)
                middlewares.append(mw)
                enabled.append(clspath)
            except NotConfigured as e:
                if e.args:
                    logger.warning(
                        "Disabled %(clspath)s: %(eargs)s",
                        {"clspath": clspath, "eargs": e.args[0]},
                        extra={"crawler": crawler},
                    )

        logger.info(
            "Enabled %(componentname)ss:\n%(enabledlist)s",
            {
                "componentname": cls.component_name,
                "enabledlist": pprint.pformat(enabled),
            },
            extra={"crawler": crawler},
        )
        return cls(*middlewares)

    @classmethod
    def from_crawler(cls, crawler: Crawler) -> Self:
        return cls.from_settings(crawler.settings, crawler)

    def _add_middleware(self, mw: Any) -> None:
        if hasattr(mw, "open_spider"):
            self.methods["open_spider"].append(mw.open_spider)
        if hasattr(mw, "close_spider"):
            self.methods["close_spider"].appendleft(mw.close_spider)

    def _process_parallel(self, methodname: str, obj: Any, *args: Any) -> Deferred:
        methods = cast(Iterable[Callable], self.methods[methodname])
        return process_parallel(methods, obj, *args)

    def _process_chain(self, methodname: str, obj: Any, *args: Any) -> Deferred:
        methods = cast(Iterable[Callable], self.methods[methodname])
        return process_chain(methods, obj, *args)

    def open_spider(self, spider: Spider) -> Deferred:
        return self._process_parallel("open_spider", spider)

    def close_spider(self, spider: Spider) -> Deferred:
        return self._process_parallel("close_spider", spider)

```

`MiddlewareManager` 클래스의 상속 구조를 도식화 해보면 다음과 같다.

~~~mermaid
classDiagram
direction TB
class MiddlewareManager
class DownloaderMiddlewareManager
class SpiderMiddlewareManager
class ItemPipelineManager
class ExtensionManager

MiddlewareManager  <|--  DownloaderMiddlewareManager 
MiddlewareManager  <|--  SpiderMiddlewareManager 
MiddlewareManager  <|--  ItemPipelineManager 
MiddlewareManager  <|--  ExtensionManager 

~~~


내부 구현 로직을 자세하게 분석할 필요는 없지만, 클래스들의 코드 동작을 살펴보면 다음과 같은 기능을 하고 있음을 알 수 있다.

- `MiddlewareManager`
  - 미들웨어 기능을 해야 하는 컴포넌트의 뼈대 클래스
  - `Settings` 객체로부터 자신에게 필요한 미들웨어들을 추출하여 생성하고, 이를 `Twisted` 프레임워크를 사용하여 병렬/연쇄적으로 처리하는 기능을 갖추고 있음
- `ItemPipelineManager`
  - `MiddlewareManager` 를 상속
  - Item Pipeline 관련된 기능을 처리하는 역할을 수행

### 왜 이렇게 복잡하게 처리가 필요한가

Scrapy 프레임워크를 사용하면서 Item 을 `Yield` 하면, 프레임워크가 처리해주는 작업은 여러가지가 있다.
1. Item 을 Item Pipeline으로 필터링하는 역할
2. 시그널 (이벤트) 처리
3. 로깅 처리

이와 관련된 기능은 대부분 Scrapy Engine 과 그 내부에 있는 컴포넌트 (특히, `Scraper`) 들을 거쳐 결정된다.  
**따라서, 기존 Scrapy 의 파이프라인 기능을 그대로 사용하려면 Engine 내부에 있는 컴포넌트들을 사용해야 문제 없이 처리가 가능하다.**

또한, 이 외에도 Item Pipeline 을 통과한 아이템들은 `ItemExporter` 를 거쳐 `FeedSlot` 객체에 모이게 되는데, [feed_slot_closed](https://docs.scrapy.org/en/latest/topics/signals.html#scrapy.signals.feed_slot_closed) , 와 같은 시그널을 처리하는 시그널 핸들러를 사용하면 수집이 종료된 시점에 최종적인 수집 데이터를 사용하여 API나 이벤트 큐와 통신하는 것들이 매우 용이해진다.
이런 기능을 그대로 사용하려면, 마찬가지로 수집된 Item 들을 Item Pipeline 을 통하도록 하는 것이 가장 간단하다.
이를 쉽게 가능하도록 하는 방법은 `Scraper` 를 사용하는 것이 현재까지는 가장 적절해 보인다.

### 사용 예시
다음과 같은 목표를 가지고 Spider와 Pipeline을 작성해야 한다고 가정해보자.
1. 특정한 데이터를 수집 시 DB 테이블에 Insert 한다.
2. 수집 과정에서 DB에 있는 데이터를 Update 처리한다.
3. 수집이 끝난 시점에서 모든 데이터를 DB에서 모두 읽어 Yield 한다.

위의 요구사항을 충족만 하는 최소한의 예시 프로그램을 만들어 본다.
1. `InsertItem` 을 수집 시 DB 테이블에 Insert 처리한다.
2. `OverwriteItem` 을 수집 시 데이터를 Update 처리한다.
3. 수집기는 최종적으로, `FinalItem` 만을 수집해야 한다.

간단하게 작성해 본 코드는 다음과 같다. (`SQLAlchemy`, `Scrapy` 기반)

```python
from collections.abc import Iterable
from typing import Any

from scrapy import Spider
from scrapy.exceptions import DropItem
from scrapy.http import TextResponse
from scrapy.http.request import Request
from scrapy.item import Field, Item
from sqlalchemy import Engine, create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, sessionmaker
from sqlalchemy.sql.expression import insert, select, update
from twisted.internet.defer import gatherResults


class BaseItem(Item):
    number = Field()
    status = Field()


class InsertItem(BaseItem):
    pass


class OverwriteItem(BaseItem):
    pass


class FinalItem(BaseItem):
    pass


MAX_ITEMS = 20  
BATCH_SIZE = 10


class SchemeBase(DeclarativeBase):
    pass


class TestScheme(SchemeBase):
    __tablename__ = "test"

    number: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[str] = mapped_column(nullable=False)


class TestPipeline:
    engine: Engine
    session_maker: sessionmaker

    def open_spider(self, spider):
        """Initialize the database engine and session maker"""
        self.engine = create_engine("sqlite:///test.db")
        self.session_maker = sessionmaker(bind=self.engine)

        SchemeBase.metadata.create_all(self.engine)

    def process_item(self, item, spider):
        """Insert or update the item in the database"""

        with self.session_maker() as session:
            if isinstance(item, InsertItem):
                session.execute(
                    insert(TestScheme).values(
                        number=item["number"],
                        status=item["status"],
                    ),
                )
                session.commit()
                raise DropItem(f"Inserted item {item['number']}")
            elif isinstance(item, OverwriteItem):
                session.execute(
                    update(TestScheme)
                    .where(TestScheme.number == item["number"])
                    .values(status=item["status"]),
                )
                session.commit()
                raise DropItem(f"Overwritten item {item['number']}")

        return item

    def close_spider(self, spider):
        """
        Read all items from the database and convert them to FinalItem,
        then pass them to pipeline for further processing
        """

        dfd_list = []

        for i in range(0, MAX_ITEMS, BATCH_SIZE):
            with self.session_maker() as session:
                items = (
                    session.execute(
                        select(TestScheme)
                        .order_by(TestScheme.number)
                        .offset(i)
                        .limit(BATCH_SIZE),
                    )
                    .scalars()
                    .all()
                )

            conv_items = [
                FinalItem(
                    number=item.number,
                    status="final",
                )
                for item in items
            ]
            cb_dfd = spider.crawler.engine.scraper.handle_spider_output(
                conv_items,
                None,
                None,
                spider,
            )
            dfd_list.append(cb_dfd)

        return gatherResults(dfd_list)


class TestSpider(Spider):
    name = "test_spider"
    start_urls = ["http://example.com"]

    custom_settings: dict[str, Any] = {
        "ITEM_PIPELINES": {
            TestPipeline: 1000,
        },
        "FEEDS": {
            "test.jl": {
                "format": "jsonlines",
                "overwrite": True,
            },
        },
        "DOWNLOAD_DELAY": 2,  # prevent ddos
    }

    def parse(
        self,
        response: TextResponse,
        **kwargs: Any,
    ) -> Iterable[Item | Request]:
        for i in range(0, MAX_ITEMS, BATCH_SIZE):
            for j in range(i, i + BATCH_SIZE):
                yield InsertItem(**{"number": j, "status": "insert"})

            yield Request(
                url="http://example.com",
                callback=self.overwrite,
                cb_kwargs={"number": i},
                dont_filter=True,
            )

    def overwrite(self, response: TextResponse, number: int) -> Iterable[Item]:
        for i in range(number, number + BATCH_SIZE):
            yield OverwriteItem(**{"number": i, "status": "overwrite"})

```

각 컴포넌트 별로 수행하는 기능은 다음과 같다.

- Spider (`TestSpider`)
  - Request 또는 Item 을 Yield 하는 역할만 수행
- Item Pipeline (`TestPipeline`)
  - DB (`SQLAlchemy` 기반의 `SQLite` 엔진, 세션) 관리
  - Item 유형에 따라 분기 처리
    - `InsertItem` 인 경우 Insert 후 Item Drop
    - `OverwriteItem` 인 경우 Update 후 Item Drop
    - 그 외의 경우 item return
  - `close_spider` 호출 시 DB 에 저장한 데이터를 모두 파이프라인으로 전송

`runspider` 등의 명령어로 실행 시 다음과 같이 로그가 남는다.

```log
2024-06-13 02:37:55 [scrapy.utils.log] INFO: Scrapy 2.11.2 started (bot: scrapybot)
2024-06-13 02:37:55 [scrapy.utils.log] INFO: Versions: lxml 5.2.2.0, libxml2 2.12.6, cssselect 1.2.0, parsel 1.9.1, w3lib 2.2.0, Twisted 24.3.0, Python 3.11.9 (main, Apr 15 2024, 18:24:23) [Clang 17.0.6 ], pyOpenSSL 24.1.0 (OpenSSL 3.2.2 4 Jun 2024), cryptography 42.0.8, Platform Linux-5.15.0-102-generic-x86_64-with-glibc2.35
2024-06-13 02:37:55 [scrapy.addons] INFO: Enabled addons:
[]
2024-06-13 02:37:55 [scrapy.utils.log] DEBUG: Using reactor: twisted.internet.epollreactor.EPollReactor
2024-06-13 02:37:55 [scrapy.extensions.telnet] INFO: Telnet Password: c1ff0f3b318422a0
2024-06-13 02:37:55 [scrapy.middleware] INFO: Enabled extensions:
['scrapy.extensions.corestats.CoreStats',
 'scrapy.extensions.telnet.TelnetConsole',
 'scrapy.extensions.memusage.MemoryUsage',
 'scrapy.extensions.feedexport.FeedExporter',
 'scrapy.extensions.logstats.LogStats']
2024-06-13 02:37:55 [scrapy.crawler] INFO: Overridden settings:
{'DOWNLOAD_DELAY': 2,
 'REQUEST_FINGERPRINTER_IMPLEMENTATION': '2.7',
 'SPIDER_LOADER_WARN_ONLY': True}
2024-06-13 02:37:55 [scrapy.middleware] INFO: Enabled downloader middlewares:
['scrapy.downloadermiddlewares.offsite.OffsiteMiddleware',
 'scrapy.downloadermiddlewares.httpauth.HttpAuthMiddleware',
 'scrapy.downloadermiddlewares.downloadtimeout.DownloadTimeoutMiddleware',
 'scrapy.downloadermiddlewares.defaultheaders.DefaultHeadersMiddleware',
 'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware',
 'scrapy.downloadermiddlewares.retry.RetryMiddleware',
 'scrapy.downloadermiddlewares.redirect.MetaRefreshMiddleware',
 'scrapy.downloadermiddlewares.httpcompression.HttpCompressionMiddleware',
 'scrapy.downloadermiddlewares.redirect.RedirectMiddleware',
 'scrapy.downloadermiddlewares.cookies.CookiesMiddleware',
 'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware',
 'scrapy.downloadermiddlewares.stats.DownloaderStats']
2024-06-13 02:37:55 [scrapy.middleware] INFO: Enabled spider middlewares:
['scrapy.spidermiddlewares.httperror.HttpErrorMiddleware',
 'scrapy.spidermiddlewares.referer.RefererMiddleware',
 'scrapy.spidermiddlewares.urllength.UrlLengthMiddleware',
 'scrapy.spidermiddlewares.depth.DepthMiddleware']
2024-06-13 02:37:55 [scrapy.middleware] INFO: Enabled item pipelines:
[<class 'test_spider.TestPipeline'>]
2024-06-13 02:37:55 [scrapy.core.engine] INFO: Spider opened
2024-06-13 02:37:55 [scrapy.extensions.logstats] INFO: Crawled 0 pages (at 0 pages/min), scraped 0 items (at 0 items/min)
2024-06-13 02:37:55 [scrapy.extensions.telnet] INFO: Telnet console listening on 127.0.0.1:6023
2024-06-13 02:37:56 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://example.com> (referer: None)
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 0
{'number': 0, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 1
{'number': 1, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 2
{'number': 2, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 3
{'number': 3, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 4
{'number': 4, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 5
{'number': 5, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 6
{'number': 6, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 7
{'number': 7, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 8
{'number': 8, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 9
{'number': 9, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 10
{'number': 10, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 11
{'number': 11, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 12
{'number': 12, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 13
{'number': 13, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 14
{'number': 14, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 15
{'number': 15, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 16
{'number': 16, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 17
{'number': 17, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 18
{'number': 18, 'status': 'insert'}
2024-06-13 02:37:56 [scrapy.core.scraper] WARNING: Dropped: Inserted item 19
{'number': 19, 'status': 'insert'}
2024-06-13 02:37:57 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://example.com> (referer: http://example.com)
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 0
{'number': 0, 'status': 'overwrite'}
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 1
{'number': 1, 'status': 'overwrite'}
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 2
{'number': 2, 'status': 'overwrite'}
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 3
{'number': 3, 'status': 'overwrite'}
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 4
{'number': 4, 'status': 'overwrite'}
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 5
{'number': 5, 'status': 'overwrite'}
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 6
{'number': 6, 'status': 'overwrite'}
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 7
{'number': 7, 'status': 'overwrite'}
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 8
{'number': 8, 'status': 'overwrite'}
2024-06-13 02:37:57 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 9
{'number': 9, 'status': 'overwrite'}
2024-06-13 02:37:58 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://example.com> (referer: http://example.com)
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 10
{'number': 10, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 11
{'number': 11, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 12
{'number': 12, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 13
{'number': 13, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 14
{'number': 14, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 15
{'number': 15, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 16
{'number': 16, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 17
{'number': 17, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 18
{'number': 18, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.scraper] WARNING: Dropped: Overwritten item 19
{'number': 19, 'status': 'overwrite'}
2024-06-13 02:37:59 [scrapy.core.engine] INFO: Closing spider (finished)
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 0, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 1, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 2, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 3, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 4, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 5, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 6, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 7, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 8, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 9, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 10, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 11, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 12, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 13, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 14, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 15, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 16, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 17, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 18, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.core.scraper] DEBUG: Scraped from None
{'number': 19, 'status': 'final'}
2024-06-13 02:37:59 [scrapy.extensions.feedexport] INFO: Stored jsonlines feed (20 items) in: test.jl
2024-06-13 02:37:59 [scrapy.statscollectors] INFO: Dumping Scrapy stats:
{'downloader/request_bytes': 694,
 'downloader/request_count': 3,
 'downloader/request_method_count/GET': 3,
 'downloader/response_bytes': 3021,
 'downloader/response_count': 3,
 'downloader/response_status_count/200': 3,
 'elapsed_time_seconds': 3.365235,
 'feedexport/success_count/FileFeedStorage': 1,
 'finish_reason': 'finished',
 'finish_time': datetime.datetime(2024, 6, 12, 17, 37, 59, 91639, tzinfo=datetime.timezone.utc),
 'httpcompression/response_bytes': 3768,
 'httpcompression/response_count': 3,
 'item_dropped_count': 40,
 'item_dropped_reasons_count/DropItem': 40,
 'item_scraped_count': 20,
 'log_count/DEBUG': 24,
 'log_count/INFO': 11,
 'log_count/WARNING': 40,
 'memusage/max': 8797593600,
 'memusage/startup': 8797593600,
 'request_depth_max': 1,
 'response_received_count': 3,
 'scheduler/dequeued': 3,
 'scheduler/dequeued/memory': 3,
 'scheduler/enqueued': 3,
 'scheduler/enqueued/memory': 3,
 'start_time': datetime.datetime(2024, 6, 12, 17, 37, 55, 726404, tzinfo=datetime.timezone.utc)}
2024-06-13 02:37:59 [scrapy.core.engine] INFO: Spider closed (finished)

```

수집된 `jsonlines` 파일의 결과를 살펴보면 다음과 같다.

```jsonlines
{"number": 0, "status": "final"}  
{"number": 1, "status": "final"}  
{"number": 2, "status": "final"}  
{"number": 3, "status": "final"}  
{"number": 4, "status": "final"}  
{"number": 5, "status": "final"}  
{"number": 6, "status": "final"}  
{"number": 7, "status": "final"}  
{"number": 8, "status": "final"}  
{"number": 9, "status": "final"}  
{"number": 10, "status": "final"}  
{"number": 11, "status": "final"}  
{"number": 12, "status": "final"}  
{"number": 13, "status": "final"}  
{"number": 14, "status": "final"}  
{"number": 15, "status": "final"}  
{"number": 16, "status": "final"}  
{"number": 17, "status": "final"}  
{"number": 18, "status": "final"}  
{"number": 19, "status": "final"}
```

우리가 원하던 최종 상태의 Item 만을 수집하는 것을 볼 수 있다.

## 결론
- Scrapy 에서 수집 종료 시점(`spider_closed` 시그널이 트리거 된 후) 에는 Item 을 `Yield` 할 수 없다.
  - 수집 종료 시점에 이를 처리하고자 한다면, `Engine` 내부에 숨겨져 있는 `Scraper` 클래스를 통해 이를 처리할 수 있다.
    - `Spider` 를 기반으로 이를 하고자 한다면 `spider.crawler.engine.scraper.handle_spider_output` 을 Item 목록에 적용하고, 적용된 `Deferred` 객체를 반환하면 된다.
- `Scraper` 클래스가 Item Pipeline 과 관련된 전반적인 생애 주기를 담당한다.
  - `Scraper` 클래스는 `ItemPipelineManager` 클래스를 사용하여 아이템 파이프라인과 Spider 에서 `Yield` 처리 된 Item 의 핸들링까지 모두 처리한다.
- 기존 컴포넌트, 시그널 핸들러, `Feed Exports` 와 관련된 기능을 사용하면서 Item 처리를 하려는 경우 Item Pipeline 을 거쳐야 하며, 이 또한 `Scraper` 클래스를 사용하여 처리할 수 있다.