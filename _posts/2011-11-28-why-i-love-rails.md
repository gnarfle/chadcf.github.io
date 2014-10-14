---
layout: post
title: Why I love rails
created: 1322508735
---
<code>
has_many :sales, :foreign_key => :CardID do
  def most_recent
    first(:order => "DateSold DESC")
  end
  def active
    first(:conditions => ["EffectiveDate <= :date AND ExpirationDate >= :date", {:date => Time.now}])
  end
end
</code>

There you go. 

@object.sales = array of all sales
@object.sales.most_recent = most recent sale
@object.sales.active = sale with a date range that contains today

It's like these guys thought of the problems people need to solve, and decided to make them easy. What an idea!
