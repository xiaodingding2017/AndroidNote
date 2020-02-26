okio高效方便之处
1、它对数据进行了分块处理(Segment)，这样在大数据IO的时候可以以块为单位进行IO，这可以提高IO的吞吐率
2、它对这些数据块使用链表来进行管理，这可以仅通过移动指针就进行数据的管理，而不用真正的处理数据，而且对扩容来说十分方便.
3、闲置的块进行管理，通过一个块池(SegmentPool)的管理，避免系统GC和申请byte时的zero-fill。其他的还有一些小细节上的优化，比如如果你把一个UTF-8的String转化为ByteString，ByteString会保留一份对原来String的引用，这样当你下次需要decode这个String时，程序通过保留的引用直接返回对应的String，从而避免了转码过程。
4、他为所有的Source、Sink提供了超时操作，这是在Java原生IO操作是没有的。
5、okio它对数据的读写都进行了封装，调用者可以十分方便的进行各种值(Stringg,short,int,hex,utf-8,base64等)的转化。
