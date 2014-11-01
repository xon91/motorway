motorway
========

Motorway is a real-time data pipeline, much like Apache Storm - but made in Python :-) We use it over at Plecto and we're really happy with it - but we're continously developing it. The reason why we started this project was that we wanted something similar to Storm, but without the following issues:

- No need to "upload" topologies
- Possibility to work tigthly with our python codebase
- "Cloud compatible" - should be able to run in AWS Auto Scaling Groups. No manual setup required for scaling and no external requirements such as Zookeeper that also do not run very nice in the Auto Scaling Groups.

Motorway re-implemented the same [algorithm to store message state](https://storm.incubator.apache.org/documentation/Acking-framework-implementation.html) as Apache Storm, which is brilliant. 

Unlike with Storm where you submit a topology to an existing cluster, with Motorway you simply add a new node with the new code and take down the other afterwards. Motorway does not (currently) communicate across nodes, so you need something like Amazon Kinesis (included), SQS (included) or Kafka to keep track of the incoming data.

Word Count Example
==================

```python
class WordRamp(Ramp):
    sentences = [
        "Oak is strong and also gives shade.",
        "Cats and dogs each hate the other.",
        "The pipe began to rust while new.",
        "Open the crate but don't break the glass.",
        "Add the sum to the product of these three.",
        "Thieves who rob friends deserve jail.",
        "The ripe taste of cheese improves with age.",
        "Act on these orders with great speed.",
        "The hog crawled under the high fence.",
        "Move the vat over the hot fire.",
    ]

    def next(self):
        yield Message(uuid.uuid4().int, self.sentences[random.randint(0, len(self.sentences) -1)])
        
class SentenceSplitIntersection(Intersection):
    def process(self, message):
        for word in message.content.split(" "):
            yield Message.new(message, word, grouping_value=word)
        message.ack()


class WordCountIntersection(Intersection):
    def __init__(self):
        self._count = defaultdict(int)
        super(WordCountIntersection, self).__init__()

    @batch_process(wait=2, limit=500)
    def process(self, messages):
        for message in messages:
            self._count[message.content] += 1
            message.ack()
        print self._count

class WordCountPipeline(Pipeline):
    def definition(self):
        self.add_ramp(WordRamp, 'sentence')
        self.add_intersection(SentenceSplitIntersection, 'sentence', 'word')
        self.add_intersection(WordCountIntersection, 'word')


WordCountPipeline().run()
```

License
=======
   Copyright 2014 Plecto ApS

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
