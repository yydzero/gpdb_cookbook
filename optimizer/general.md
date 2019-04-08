# Optimizer

## disable xfrom to save planning time when have many joins

    SELECT disable_xform('CXformJoinAssociativity'); // session specific
