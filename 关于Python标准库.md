
<b>
待补

下载图片时gzip碰到一个bug:

      File "/Users/hipponensis/myprojects/crawlertool/douban_movies.py", line 136, in crawler_image
        image_data = g.read()
      File "/Users/hipponensis/.pyenv/versions/3.5.1/lib/python3.5/gzip.py", line 274, in read
        return self._buffer.read(size)
      File "/Users/hipponensis/.pyenv/versions/3.5.1/lib/python3.5/gzip.py", line 461, in read
        if not self._read_gzip_header():
      File "/Users/hipponensis/.pyenv/versions/3.5.1/lib/python3.5/gzip.py", line 409, in _read_gzip_header
        raise OSError('Not a gzipped file (%r)' % magic)

看了[这个](http://stackoverflow.com/questions/4928560/how-can-i-work-with-gzip-files-which-contain-extra-data), 了解到原因是内置模块设计不合理。

又看了[这个](http://stackoverflow.com/questions/13137817/how-to-download-image-using-requests)和[这个](http://stackoverflow.com/questions/16813267/python-gzip-refuses-to-read-uncompressed-file), 最后使用shutil.copyfileobj问题解决, 但数据未压缩比较耗流量, 等下次抽空优化。
