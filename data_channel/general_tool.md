# export data & import data

## copy data with compression

    \copy tmp_q from program 'zcat tmp_q.gz'
    \copt tmp_r from program 'cat tmp_r.gz.* |zcat'
