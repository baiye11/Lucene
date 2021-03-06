<h1>基于Lucene搜索引擎的邮件检索</h1>

<h3>一、问题描述</h3>

现在用户提供了一份数据集，其内容为安然公司150位用户50万封电子邮件。用户希望通过我的程序，可以实现按照收件人、发件人、标题、内容等分类进行对邮件的检索。针对这个任务，我的程序需要根据用户的常用检索类别以及其他需求，对50万封电子邮件进行的分类，相当于可以理解成将50万封电子邮件插入到以不同的收发件人、标题、内容为头指针的链表中，以实现更高效的信息检索。Lucene包提供了非常多有用的类和函数，可以帮助实现这个任务。

 

<h3>二、实验环境介绍</h3>

为了完成任务，我在项目中导入了lucene-analyzers-common-8.2.0.jar，lucene-core-8.2.0.jar，lucene-queryparser-8.2.0.jar三个核心的lucene包。为了更好地记录程序出现的错误，我还在项目中导入了log4j-api-2.12.1.jar，log4j-core-2.12.1.jar，log4j-web-2.12.1.jar三个记录错误信息的日志类第三方包。程序是在自己的手提电脑上通过IntelliJ Idea进行编译和运行。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps1.jpg) 

<p align="middle">图2-1</p>

 

<h3>三、实现思路</h3>

50万封电子邮件是一个有一定数量级的数据集，因此通过普通的方法对电子邮件进行标记、存储都不实际，因此我采用Lucene的类与方法对数据集进行处理。

首先，我对用户所需要的收件人、发件人、标题、内容等特征项进行了分析，确定了文件名，文件路径，邮件的主题，邮件的部分内容，收发件人的电子邮箱，收发件人的姓名这几个索引。其中除文件名和文件路径以外，其余的索引都以TextField的域建立，以支持在检索时只输入索引的字段中的一部分，仍可以检索到完整字段对应的邮件这一功能。而对于文件名，采取的必须是输入准确的完整的文件名才可能返回正确的邮件信息；对于文件路径，采用的是StoredField的域类型，目的是为了存储文件的路径，其作为关键字段进行检索的功能并不重要。

其次，对于每一个索引分类，我为他们写了各自的函数完成索引关键字段的获取。经过观察，我发现每一份文件在最开头都会将邮件的基本信息详实的进行记录。因此我可以通过文件的I/O操作读入文件中的内容，以关键字识别的方式，获取文件中记录邮件的收发件人和主题、具体内容等信息。通过对字符串的简单操作，完成内容的处理，使每一份信息都规范化，有利于检索时的关键字段处理。

然后，我专门写了一个对给定源文件中的所有文件的遍历并建立索引的函数。由于源文件夹下仍有许多文件夹，因此我采用了队列的思想，将源文件夹、子文件夹依次入队，每个文件夹在完成所有文件的读入、信息处理或者所有文件夹的信息存储之后，则该文件夹出队列。通过这个方法实现了遍历一个给定路径下所有的文件。

最后，建立好索引后，我通过在主函数中请用户选择需要查询的类别，输入需要查询的内容，以及选择查询的模式，实现模糊查询、精确查询和全文查询（即所有50万封邮件）三种查询方式，完成查询后返回命中的结果并打印前十个相关性排序最高的邮件的部分信息。

 

<h3>四、具体实现</h3>

<h5>4.1 建立索引的类别</h5>

根据用户的需求和实际应用场景，我将索引分成了文件名、文件路径、邮件主题、邮件部分内容、邮件的收发件人邮箱以及收发件人名字这几类，并根据这些索引的具体应用设置了不同属性的域。如文件名则采用了StringField域构建，邮件主题、内容、收发件人等则采用了TextField域构建，目的是为了给用户更好的体验，即使忘记了准确的信息，也可以匹配到相关度高的邮件信息。图4-1、4-2、4-3是所有的我建立索引的代码。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps2.jpg) 

<p align="middle">图4-1 文件名称、文件路径和邮件内容的索引建立</p>



![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps3.jpg) 

<p align="middle">图4-2 发件人邮箱和发件人姓名的索引建立</p>



![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps4.jpg) 

<p align="middle">图4-3 收件人邮箱、收件人姓名、邮件主题的索引建立</p>

 

<h5>4.2 建立索引关键字段生成的函数</h5>

索引的关键字段生成需要基于邮件内容。经过随机抽查，我发现每一封邮件在开头都会有相似的格式记录邮件的收发件人、时间、主题等信息，并且这些信息的开头往往有统一的标志性字符串，如主题是“Subject：”，收件人邮箱是“From：”，收件人姓名是“X-From：”，等。根据这样的特点，我就可以通过I/O操作进行文件的读取，将每一行的内容记录到字符串中，对该字符串进行处理。这其中包括判断该行是否包括关键信息字段，判断该行的与前几行的关系以提取一整段的邮件内容等。以获取收件人和收件人邮箱为例（图4-4、4-5）。程序对文件一行一行读取，每一行都进行判断，该行是否含有关键词“From：”，如果有，则将该行从“：”后一个位置截取字符串，并去除字符串前后的空格。去除空格是因为我发现文本中每一个关键词的冒号后都有一个空格，为更好地查询，于是我对字符串进行了这样的处理（如箭头所示）。由于可能出现没有收发件人的情况，但基于种种原因，我认为返回没有内容的字符串比直接返回一个空指针更有效，因此在函数返回值时进行了处理，如图4-5最后一个else的代码块所示。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps5.png 

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps7.jpg) 

<p align="middle">图4-4、4-5</p>



值得一提的是，由于邮件存在多次来往的情况，同时有的邮件中记录的收发件人的邮箱并不完全符合邮箱的格式，因此我专门设置了一个判断邮箱格式是否正确的函数，如果邮箱格式不正确，则会通过log4j的日志功能进行记录，不影响程序的正常运行，但事后应该需要针对这些邮箱格式不正确的邮件进行进一步细化的信息处理。

由于判断邮件的函数有比较多的if判断和while循环，截图效果欠佳，因此不在报告中展示。主要判断的思想是，先判断获取的邮件字符串中是否有“@”，且“@”不在字符串的第一个位置，然后判断“@”后的最后一个“.”后是否有“com”字符串。接着还需要判断字符串中是否有出现两个“.”之间有没有内容以及“.”和“@”之间有没有内容。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps8.jpg) 

<p align="middle">图4-6 从文件中读取信息以建立索引的文件内容处理类</p>

 

<h5>4.3建立遍历根文件夹下的所有文件的函数</h5>

由于用户提供的根文件夹下往往还有很多子文件夹，因此我专门写了一个函数，进行文件和文件夹的处理。我采用了队列的思想，每一个文件被传入函数时，需要先被判断是文件还是文件夹，如果是文件则直接对其进行信息的提取和处理，如果是根文件夹的话则会加入到一个队列中，当这个文件夹下的所有子文件夹和子文件都已经入队或者完成处理后，根文件夹会出队。思路和二叉树的层序遍历的想法接近。以上通过队列思想实现的对所有文件的遍历功能是这个函数的返回值和建立索引的主要函数相配合使用的。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps9.jpg) 

<p align="middle">图4-3 建立索引的主要函数</p>



![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps10.jpg)

<p align="middle">图4-4遍历和建立索引的具体实现函数</p>



图4-3是建立索引的外包装函数，其中while语句即实现队列的思想的文件遍历，用Directory这个链表去记录文件夹的名字，如果是文件夹则入队，将这个文件夹的文件遍历完后该文件夹出队，直到链表里不再有文件夹，就意味着所有的文件都被访问过了。图4-4是具体的将文件夹入队，对文件进行索引建立的内部实现函数，对文件夹建立索引是if(file.isFile()){}下的代码块，文件夹入队是else if(file.isDirectory()){}下的代码块。具体文件夹的索引建立见4.1建立索引的类别。

 

<h5>4.4用户的信息检索</h5>

完成对所有邮件的索引建立后，我在主方法中为用户提供检索项的选择以及检索方法的选择，并允许用户选择是否将搜索到的信息打印到文件中。选择检索方法体现于在主函数中询问用户是否确定自己输入的信息是完全正确的，如果用户反馈为是的话，那么程序将调用TermQuery子类按精准词条的方式进行检索，如果用户反馈为否的话，那么程序将调用FuzzyQuery子类按模糊搜索的方式进行内容的检索。从搜索结果可以看出，两种方式的命中邮件数是不相同的。当然如果用户没有给出正确的反馈的话，那么程序将按照特定分析器的检索，实现对全部邮件的词语的检索，即QueryParser。这种检索是通过特定的分词器将所有文件的内容进行划分后，根据需要检索的内容进行完整匹配。这种方法尽管只是多增加几行代码，但有利于不同用户需要的小范围精确检索和大范围的模糊检索的需求。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps11.jpg) 

 <p align="middle">图4-5</p>



<h3>五、结果展示</h3>

以下图片展示的是用户进入程序界面之后通过引导进行输入后完成的检索结果。其中图5-1、5-2是查询文件名为“100_”的邮件的检索结果，由于控制台输出有限，因此我只选取了得分最高的前十个邮件进行信息的输出，以下只截取了前三封邮件的信息。图5-3展示的是检索邮件内容的结果，可以看到邮件的内容是包含所输入的内容“day”，这里就发挥了TextField域既可以建立索引又完成分词的作用。图5-4、5-5是对模糊查询功能实现的很好地反馈，我需要查找的文件名是“1_”，但我并不确定我的文件名内容，也就是我这时候希望调用模糊查询的FuzzyQuery，可以看出，打印的信息中有文件名为“10_”“11_”“14_”的结果。图5-4用FuzzyQuery查询出来的命中结果有43542条，而图5-6用TermQuery查询出来的命中结果只有3311条，这就很好地实现了我希望的根据不同用户对数据精确度需求进行的信息检索。图5-7是输出到文件的效果。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps12.jpg) 

<p align="middle">图5-1</p>

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps13.jpg) 

<p align="middle">图5-2</p>

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps14.jpg) 

<p align="middle">图5-3</p>

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps15.jpg) 

<p align="middle">图5-4</p>

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps16.jpg) 

<p align="middle">图5-5</p>

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps17.jpg) 

<p align="middle">图5-6</p>

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps18.jpg) 

<p align="middle">图5-7</p>



<h3>六、改进和提升</h3>

<h5>6.1收件人的索引建立改进</h5>

由于在很多的邮件中往往出现收件人都不止一个，因此在建立索引时，往往需要对同一封邮件的同一个索引“EmailOfReceiver”添加很多关键字段。但这会导致在获取索引“EmailOfReceiver”时无法返回关键字段。因此我采取的方法是将该索引的域设置成TextField，即可以进行分词的域，然后将所有的收件人信息合并成一个字符串添加到索引“EmailOfReceiver”中。这是我在耗费大量时间调试之后尝试的一个曲线救国的方法。

这看上去非常不错，至少可以建立好收件人索引，并没有异常错误的返回需要的信息，但这会导致每一封邮件都有一个对应的收件人标签，而两封邮件的标签只有完全一样，即收件人的数量和收件人的邮箱完全一致，那么两封邮件才可能被放到同一个索引的关键字段下。本来一个收件人邮箱可以连接很多不同的邮件，即使这些邮件可能不止一个收件人，且收件人不尽相同，现在这样处理的话导致索引的关键字段会有很多重复的内容。

但由于作业时间有限，目前还没有想到更好的办法，只能牺牲建立索引的空间。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps19.jpg) 

<p align="middle">图6-1</p>

 

<h5>6.2模糊搜索的匹配度设置和反馈</h5>

我希望在建立模糊搜索时，可以根据用户的需求进行匹配度的限制。这是可以实现的，但由于时间关系暂时还没有完善这部分的代码。而由于用户选用了模糊搜索，那我希望反馈一个信息，告诉用户我们搜索出来的结果跟你的要搜索的内容的匹配度有多少，经过资料的查找，目前采用的是topdocs的score的分数，这是lucene包内对每一个建立索引的文件进行的相关度优先级排序的得分。由于暂时没发现更好地展示相关匹配度的方法，因此暂时使用这个分数给予用户反馈。

 

<h5>6.3多种搜索方式的设置</h5>

设置多种搜索方式并非事出偶然。因为在建好索引后，总有一部分的搜索没有结果，经过分析发现，是因为TermQuery是一个比较精确的词条搜索方法，这个方法要求用户输入非常精确的信息。如果对关键字段的数据处理不当的话，可能用户在输入时少敲了一个空格都会导致无法检索的问题。一开始我没有很好地理解TermQuery的作用，因此我花费了大量的时间进行调试而颗粒无收。后来发现了这个问题之后，我决定提供多种搜索方式，以达到不同用户根据其掌握的信息准确度进行不同精度的查询这样一个效果。

 

<h5>6.4 log4j包的使用与异常记录</h5>

由于在程序编写的过程中，我遇到了收件人明明建立了索引但无法按预期返回的情况，因此我配置了log4j包以更好地记录我想要的info或者出现的异常。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps20.jpg) 

<p align="middle">图6-2记录不合规格的收件人邮箱地址的信息示例</p>

 

<h5>6.5 对已存在索引的处理</h5>

在索引已经建立之后，当我有修改程序的行为后，容易造成新索引和原索引不匹配的错误（图6-3）。因此我加了这两条语句，以减少新索引和原索引的冲突（图6-4）。另外我还发现了另一种更简洁的对新加入索引管理的语句，和后来上机课的时候助教提到了语句一样（图6-4的注释代码）。

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps21.jpg) 

<p align="middle">图6-3</p>

![img](file:///C:\Users\麦子\AppData\Local\Temp\ksohtml11104\wps22.jpg) 

<p align="middle">图6-4</p>

 
