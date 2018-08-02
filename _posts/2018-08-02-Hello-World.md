---
title: Hello World
---

<center><h1>Hello World</h1></center>
<center><font size="1">test for highlight</font></center>

<pre>
class Solution(object):
    def containsNearbyDuplicate(self, nums, k):
        """
        :type nums: List[int]
        :type k: int
        :rtype: bool
        """
		if nums == []:
            return False
        if k == 0:
            return False
        dict_ = {}
        for i in range(len(nums)):
            if nums[i] not in dict_:
                dict_[nums[i]] = i
            elif i - dict_[nums[i]] <= k:
                return True
            else:
                dict_[nums[i]] = i
        return False
</pre>