---
layout: post
title:  "Encounter Hard: Design Search Autocomplete System"
date:   2020-03-18 15:59:53 -0700
categories: algorithm
tag: [leetcode, trie, heap, hard]
---

## Algorithm
#### 问题
```
 642. Design Search Autocomplete System

 Share
 Design a search autocomplete system for a search engine. Users may input a sentence \
 (at least one word and end with a special character '#').
 For each character they type except '#', you need to return the top 3 historical hot sentences that have prefix the
 same as the part of sentence already typed. 
 Here are the specific rules:

 The hot degree for a sentence is defined as the number of times a user typed the exactly same sentence before.
 The returned top 3 hot sentences should be sorted by hot degree (The first is the hottest one). 
 If several sentences have the same degree of hot, you need to use ASCII-code order (smaller one appears first).
 If less than 3 hot sentences exist, then just return as many as you can.
 When the input is a special character, it means the sentence ends, 
 and in this case, you need to return an empty list.
 Your job is to implement the following functions:

 The constructor function:

 AutocompleteSystem(String[] sentences, int[] times): This is the constructor. 
 The input is historical data. Sentences is a string array consists of previously typed sentences. 
 Times is the corresponding times a sentence has been typed.
 Your system should record these historical data.

 Now, the user wants to input a new sentence. The following function will provide the next character the user types:

 List<String> input(char c): The input c is the next character typed by the user.
 The character will only be lower-case letters ('a' to 'z'), blank space (' ') or a special character ('#').
 Also, the previously typed sentence should be recorded in your system.
 The output will be the top 3 historical hot sentences that have prefix the same as the part of sentence already typed.


 Example:
 Operation: AutocompleteSystem(["i love you", "island","ironman", "i love leetcode"], [5,3,2,2])
 The system have already tracked down the following sentences and their corresponding times:
 "i love you" : 5 times
 "island" : 3 times
 "ironman" : 2 times
 "i love leetcode" : 2 times
 Now, the user begins another search:

 Operation: input('i')
 Output: ["i love you", "island","i love leetcode"]
 Explanation:
 There are four sentences that have prefix "i". Among them, "ironman" and "i love leetcode" have same hot degree.
 Since ' ' has ASCII code 32 and 'r' has ASCII code 114,
 "i love leetcode" should be in front of "ironman".
 Also we only need to output top 3 hot sentences, so "ironman" will be ignored.

 Operation: input(' ')
 Output: ["i love you","i love leetcode"]
 Explanation:
 There are only two sentences that have prefix "i ".

 Operation: input('a')
 Output: []
 Explanation:
 There are no sentences that have prefix "i a".

 Operation: input('#')
 Output: []
 Explanation:
 The user finished the input, the sentence "i a" should be saved as a historical sentence in system.
 And the following input will be counted as a new search.

 Note:
 The input sentence will always start with a letter and end with '#', and only one blank space will exist between two words.
 The number of complete sentences that to be searched won't exceed 100.
 The length of each sentence including those in the historical data won't exceed 100.
 Please use double-quote instead of single-quote when you write test cases even for a character input.
 Please remember to RESET your class variables declared in class AutocompleteSystem,
 as static/class variables are persisted across multiple test cases. Please see here for more details.
```
标榜Hard, 面试热门户，输入随机所以无法用bucket sort来做。  
搜索时是一个字母一搜~~谷歌体验~~，需要cache住每次递增的prefix, 我没有有幸看过真正的free type搜索bar是如何实现的。

有关阅读 [Trie的总结](https://tonyhuang.cloud/algorithm/2020/03/18/trie-types/)
### Trie Node complete version
```java
public class AutoCompTrieNode {
    Map<Character, AutoCompTrieNode> children;
    Map<String, Integer> counts;
    boolean isWord;
    public AutoCompTrieNode() {
        children = new HashMap<Character, AutoCompTrieNode>();
        counts = new HashMap<String, Integer>();
        isWord = false;
    }
}
```
为Auto complete特供的Trie Node, 每个Node记录了这层含有char的单词出现的次数。



要点
- 建立Trie的时候需要在每一层的每一个node保存含有目前char的单词的出现次数
  - e.g. apple 出现5次
    - node `a` 需要保存<apple, 5>
    - 最后一个node `e` 也需要保存<apple, 5>
  - 如此方便在搜索字节变长的时候能快速得到单词而不用临时组装
- 输出Top K就是老朋友Max Heap了， 其余没有什么难度

```java
public class DesignSearchAutocompleteSystem {
	AutoCompTrieNode root;
	String prefixCache;

	/**
	 * - Based by trie tree, As the input is dynamic, bucket sort cannot be used,
	 *   in the case when a new input is tried multiple times, the bucket will experience out of bounce
	 *
	 * - Special character might also used, an array of ascII table each node is very costy
	 *
	 * - need to use dynamic data structure
	 *
	 * - Key idea is similar to the kth frequent item. push to the priority queue and find kth
	 * - trick is to keep a global prefix for continuous search
	 */
	public DesignSearchAutocompleteSystem(String[] sentences, int[] times) {
		root = new AutoCompTrieNode();
		prefixCache = "";

		for (int i = 0; i < sentences.length; i++) {
			build(sentences[i], times[i]);
		}
	}

	private void build(String s, int count) {
		AutoCompTrieNode curr = root;
		for (char c : s.toCharArray()) {
			AutoCompTrieNode next = curr.children.get(c);
			if (next == null) {
				next = new AutoCompTrieNode();
				curr.children.put(c, next);
			}
			curr = next;
			curr.counts.put(s, curr.counts.getOrDefault(s, 0) + count);
		}
		curr.isWord = true;
	}


	public List<String> input(char c) {
		// check # first: add current sentences to the trie, reset to the root and return empty list
		if (c == '#') {
			// occur ++
			build(prefixCache, 1);
			prefixCache = "";
			return new ArrayList<>();
		}

		// build new search
		prefixCache += c;
		AutoCompTrieNode curr = root;

		// Search for the head start
		for (char chara : prefixCache.toCharArray()) {
			AutoCompTrieNode next = curr.children.get(chara);

			if (next == null) {
				// search does not exist;
				return new ArrayList<>();
			}
			curr = next;
		}

		// Build a max heap
		PriorityQueue<Map.Entry<String, Integer>> heap = new PriorityQueue<>((a, b) -> (
				a.getValue() == b.getValue() ?
						a.getKey().compareTo(b.getKey()) :
						b.getValue() - a.getValue()));

		// push all count to max heap
		for (Map.Entry<String, Integer> entry : curr.counts.entrySet()) {
			heap.add(entry);
		}

		// build result
		List<String> result = new ArrayList<String>();
		for (int i = 0; i < 3 && !heap.isEmpty(); i++) {
			result.add(heap.poll().getKey());
		}
		return result;
	}
}
```