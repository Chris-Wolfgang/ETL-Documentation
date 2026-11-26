type: fix

FixedWidth samples on the library page and the pipeline-composition, custom-parsers and compressed-streams guides now configure the extractor and loader through options records instead of `[Obsolete]` setters and constructors, and the custom-parsers guide no longer claims a `null` return falls back to the default parser.
