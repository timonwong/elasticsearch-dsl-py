.. _faceted_search:

Faceted Search
==============

The library comes with a simple abstraction aimed at helping you develop
faceted navigation for your data.

.. note::

    This API is experimental and will be subject to change. Any feedback is
    welcome.

Configuration
-------------

You can provide several configuration options (as class attributes) when
declaring a ``FacetedSearch`` subclass:

``index``
  the name of the index (as string) to search through, defaults to ``'_all'``.

``doc_types``
  list of ``DocType`` subclasses or strings to be used, defaults to
  ``['_all']``.

``fields``
  list of fields on the document type to search through. The list will be
  passes to ``MultiMatch`` query so can contain boost values (``'title^5'``),
  defaults to ``['*']``.

``facets``
  dictionary of facets to display/filter on. The key is the name displayed and
  values should be instances of any ``Facet`` subclass, for example: ``{'tags':
  TermsFacet(field='tags')``


Facets
~~~~~~

There are several different facets available:

``TermsFacet``
  provides an option to split documents into groups based on a value of a field, for example ``TermsFacet(field='catgeory')``

``DateHistogramFacet``
  split documents into time intervals, example: ``DateHistogramFacet(field="published_date", interval="day")``

``HistogramFacet``
  simmilar to ``DateHistogramFacet`` but for numerical values: ``HistogramFacet(field="rating", interval=2)``

``Rangefacet``
  allows you to define your own ranges for a numerical fiels:
  ``Rangefacet(field="comment_count", ranges=[("few", (None, 2)), ("lots", (2, None))])`` 

Advanced
~~~~~~~~

If you require any custom behavior or modifications simply override one or more
of the methods responsible for the class' functions. The two main methods are:

``search(self)``
  is responsible for constructing the ``Search`` object used. Override this if
  you want to customize the search object (for example by adding a global
  filter for published articles only).

``query(self, search)``
  adds the query postion of the search (if search input specified), by default
  using ``MultiField`` query. Override this if you want to modify the query type used.


Usage
-----

The custom subclass can be instantiated empty to provide an empty search
(matching everything) or with ``query`` and ``filters``.

``query``
  is used to pass in the text of the query to be performed. If ``None`` is
  passed in (default) a ``MatchAll`` query will be used. For example ``'python
  web'``

``filters``
  is a dictionary containing all the facet filters that you wish to apply. Use
  the name of the facet (from ``.facets`` attribute) as the key and one of the
  possible values as value. For example ``{'tags': 'python'}``.

Response
~~~~~~~~

the response returned from the ``FacetedSearch`` object (by calling
``.execute()``) is a subclass of the standard ``Response`` class that adds a
property called ``facets`` which contains a dictionary with lists of buckets -
each represented by a tuple of key, document cound and a flag indicating
whether this value has been filtered on.

Example
-------

.. code:: python

    from datetime import date

    from elasticsearch_dsl import FacetedSearch, TermsFacet, DateHistogramFacet

    class BlogSearch(FacetedSearch):
        doc_types = [Article, ]
        # fields that should be searched
        fields = ['tags', 'title', 'body']

        facets = {
            # use bucket aggregations to define facets
            'tags': TermsFacet(field='tags'),
            'publishing_frequency': DateHistogramFacet(field='published_from', interval='month')
        }

        def search(self):
            # override methods to add custom pieces
            s = super().search()
            return s.filter('range', publish_from={'lte': 'now/h'})

    bs = BlogSearch('python web', {'publishing_frequency': date(2015, 6)})
    response = bs.execute()

    # access hits and other attributes as usual
    print(response.hits.total, 'hits total')
    for hit in response:
        print(hit.meta.score, hit.title)

    for (tag, count, selected) in response.facets.tags:
        print(tag, ' (SELECTED):' if selected else ':', count)

    for (month, count, selected) in response.facets.publishing_frequency:
        print(month.strftime('%B %Y'), ' (SELECTED):' if selected else ':', count)


