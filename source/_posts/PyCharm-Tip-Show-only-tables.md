---
title: 'PyCharm Tip: Show only tables'
date: 2018-11-22 11:00:17
categories:
tags: [pycharm, datagrip, postgres]
---

If you work with PyCharm or DataGrip with a database and you want to access the table data it can be pretty annoying to get to them since you have to select the Connection -> Database -> Schema -> Tables - quite a few clicks. 

Here's how you can reduce the number of clicks:

- Click on the Cog (settings icon) in top-right corner in the Database Window in Pycharm
- Uncheck: `Group Data sources`, `Group Schema` and `Show Intermediary Nodes`
- Click on the Filter button inside the Database window and uncheck all the Objects you don't want to see such as `Role`
or `Access Method`.

You can also watch the video below on how to do it.

<iframe width="560" height="315" src="https://www.youtube.com/embed/tOgl6E_MDpY" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Good luck!