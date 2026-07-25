---
写作年份: 2023
tags:
  - ceph基础库
---
每个区间占用 16 bytes（存储 start 和 end）。每个树节点（包含区间和指针）占用大约 48 bytes，因此 interval_set 的总内存占用大约是 每个区间 64 bytes。如果 interval_set 中有 N 个区间，则总的内存占用为大约 64 * N bytes。

`src/include/interval_set.h`

```c++
template<typename T, typename Map = std::map<T,T>>
class interval_set {
public:
	using value_type = typename Map::value_type;
	using offset_type = T;
	using length_type = T;
	using reference = value_type&;
	using const_reference = const value_type&;
	using size_type = typename Map::size_type;

private:
	/*helpers*/
	auto find_inc(T start) const {
		auto p = m.lower_bound(start); // p->first >= start
		/*非空时(m.begin != m.end())，找不到>=start的元素或者找到的元素大于start，则左移后检查
		**p->first = start时，没有左移p，是否意味着不存在p--后，p->first + p->second > start的情况？*/
		if (p != m.begin() && (p == m.end() || p->first > start)) {
			--p; 
			/*如果没有找到p->first >= start的元素或者找到的p->first > start，则尝试左移p检查p->first + p->second的覆盖位置：
			**1. 如果<= 入参start，即没有超过start的位置，回到m.lower_bound返回的迭代器位置，即p为m.end或者p->first > start
			**2. 如果>入参start，则p覆盖的位置超过start，则返回p */
			if (p->first + p->second <= start) 
			{
				++p; // it doesn't.
			}
		}

		/*可能是
		**1. m.end()
		**2. p->first == start
		**3. p->first < start但 p->first + p->second > start
		**4. p->first > start
		**/
		return p; 
	}

	auto find_adj(T start) const {
		auto p = m.lower_bound(start);
		if (p != m.begin() && (p == m.end() || p->first > start)) {
			--p; 
			/*如果没有找到p->first >= start的元素或者找到的p->first > start，则尝试左移p检查p->first + p->second的覆盖位置：
			**1. 如果< 入参start，即没有达到start的位置，回到m.lower_bound返回的迭代器位置，即p为m.end或者p->first > start
			**2. 如果>=入参start，则p覆盖的位置至少达到start，则返回p */
			if (p->first + p->second < start) {
				++p; // it doesn't.
			}
		}
	
		/*可能是
		**1. m.end()
		**2. p->first == start
		**3. p->first < start但 p->first + p->second >= start
		**4. p->first > start
		**/
		return p;
	}

public:
	/*返回map中第一个元素的start位置*/
	offset_type range_start() const {
		ceph_assert(!empty());
		auto p = m.begin();
		return p->first;
	}

	/*返回map中最后一个元素 start + len到达的位置，这里应该没有被使用*/ 
	offset_type range_end() const {
		ceph_assert(!empty());
		auto p = m.end();
		p--;
		return p->first + p->second;
	}

	void insert(T val) { insert(val, 1); }

	void insert(T start, T len, T *pstart=0, T *plen=0) { ///???
		ceph_assert(len > 0);
		_size += len;


		auto p = find_adj_m(start); 
		/*没有找到符合如下条件的p
		**1. p->first == start
		**2. p->first < start 但 p->first + p->second >= start
		**3. p->first > start
		**/
		if (p == m.end()) { 
			m[start] = len; // new interval
			if (pstart)
				*pstart = start;
			if (plen)
				*plen = len;
		} else {
			if (p->first < start) { //p->first在start前面
				if (p->first + p->second != start) {
					ceph_abort(); //为何？
				}

				p->second += len; // append to end
				auto n = p;
				++n; //n移动到下一个位置

				if (pstart) 
					*pstart = p->first;
				/*如果p后面还有元素且start位置+len刚好到达n->first位置*/
				if (n != m.end() && start+len == n->first) {
					p->second += n->second;
					if (plen) 
						*plen = p->second;
					m.erase(n);
				} else { //p->first + p->second和n->first之间存在空隙
					if (plen) 
						*plen = p->second;
				}
			} else { //p->first就是start或者p->first在start后面
				if (start+len == p->first) {//p->first在start后面，start + len == p->first
					if (pstart)
						*pstart = start;
					if (plen)
						*plen = len + p->second;
					T psecond = p->second;
					m.erase(p);
					m[start] = len + psecond; // append to front，进行合并
				} else {
					ceph_assert(p->first > start+len); //p->first在start后面，且start + len无法到达p->first
					if (pstart)
						*pstart = start;
					if (plen)
						*plen = len;
					m[start] = len; // new interval, start + len和p->first之间存在空隙
				}
			}
		}
	}

	void erase(T start, T len, std::function<bool(T, T)> claim = {}) {
		auto p = find_inc_m(start);
		_size -= len;
		ceph_assert(_size >= 0);
		ceph_assert(p != m.end());
		ceph_assert(p->first <= start);

		T before = start - p->first;
		ceph_assert(p->second >= before+len);
		T after = p->second - before - len;

		if (before) {
			if (claim && claim(p->first, before)) {
				_size -= before;
				m.erase(p);
			} else {
				p->second = before; // shorten bit before
			}
		} else {
			m.erase(p);
		}

		if (after) {
			if (claim && claim(start + len, after)) {
				_size -= after;
			} else {
				m[start + len] = after;
			}
		}
	}

private:
// data
	uint64_t _size = 0; //所有元素len之和
	Map m; // map start -> len
}
```