---
写作年份: 2024
tags:
  - ceph
  - msg
---
```c++
//AsyncMessager.cc
Messenger *Messenger::create
	->new AsyncMessenger
		->processors.push_back()   :AsyncMessager.cc
		->Processor->bind()
```