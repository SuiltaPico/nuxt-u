<script setup lang="ts">
import {
  cat,
  comp,
  filter,
  push,
  repeat,
  repeatedly,
  take,
  transduce,
  add,
} from "@thi.ng/transducers";
import p5 from "p5";

type MatrixCellIndex = readonly [row: number, col: number];
type MatrixShape = readonly [w: number, h: number];

function get_size_of_shape(shape: MatrixShape) {
  return shape[0] * shape[1];
}

const shape = [300, 180] as MatrixShape;
const size_of_shape = get_size_of_shape(shape);

const cell_size = 5;

function make_random_vec2d() {
  return [
    ...transduce<number[], number[], number[][]>(
      take(shape[1]),
      push(),
      repeatedly(() => [
        ...transduce<number, number, number[]>(
          take(shape[0]),
          push(),
          repeatedly(() => ((Math.random() > 0.8) ? 1 : 0))
        ),
      ])
    ),
  ];
}

function make_clean_vec2d() {
  return [
    ...transduce<number[], number[], number[][]>(
      take(shape[1]),
      push(),
      repeatedly(() => [
        ...transduce<number, number, number[]>(
          take(shape[0]),
          push(),
          repeat(0)
        ),
      ])
    ),
  ];
}

const cells = {
  vec: make_random_vec2d(),
};

function get_offset_cell_index(
  src_index: MatrixCellIndex,
  offset: MatrixCellIndex,
  shape: MatrixShape
): MatrixCellIndex | undefined {
  const row_index = src_index[0] + offset[0];
  if (row_index < 0 || row_index >= shape[1]) return undefined;

  const col_index = src_index[1] + offset[1];
  if (col_index < 0 || col_index >= shape[0]) return undefined;

  return [row_index, col_index] as const;
}

/** 如果不检测或超出范围，则返回 -1。 */
function get_matrix_data(
  dm: number[][],
  self_in_dm_index: MatrixCellIndex,
  shape: MatrixShape,
  vec2d_index: MatrixCellIndex,
  rows_for_access: (undefined | number[])[]
) {
  return dm.map((dm_row, ri) => {
    return dm_row.map((dm_cell, ci) => {
      if (dm_cell === 0) return -1;

      const col_index = vec2d_index[1] + ci - self_in_dm_index[1];
      if (col_index < 0 || col_index >= shape[0]) return -1;

      const rfa_row = rows_for_access[ri];
      if (rfa_row === undefined) return -1;

      return rfa_row[col_index] * dm_cell;
    });
  });
}

function calc_next_frame(vec: number[][]) {
  // 1: check cell
  // const detection_matrix = [
  //   [1, 1, 0, 1, 1],
  //   [1, 1, 0, 1, 1],
  //   [1, 1, 0, 1, 1],
  //   [1, 1, 0, 1, 1],
  //   [1, 1, 0, 1, 1],
  // ];
  const detection_matrix = [
    [0.103, 0.413, 0.72, 0.413, 0.103],
    [0.413, 1.65, 2.68, 1.65, 0.413],
    [0.72, 2.68, 0, 2.68, 0.72],
    [0.413, 1.65, 2.68, 1.65, 0.413],
    [0.103, 0.413, 0.72, 0.413, 0.103],
  ];
  // const detection_matrix = [
  //   [1, 1, 0, 1],
  //   [1, 0, 1, 1],
  //   [1, 1, 0, 1],
  // ];

  // const self_in_dm_index = [1, 1] as const;
  const self_in_dm_index = [2, 2] as const;

  const new_frame = make_clean_vec2d();

  const ruler = (
    dm_data: number[][],
    new_frame: number[][],
    new_frame_row: number[],
    shape: MatrixShape,
    vec2d_index: MatrixCellIndex
  ) => {
    const num = transduce(
      comp(
        cat(),
        filter((it) => it > 0)
      ),
      add(),
      dm_data
    );

    // if (
    //   (num === 3 || num === 4 || num === 5) &&
    //   vec[vec2d_index[0]][vec2d_index[1]] > 0
    // ) {
    //   new_frame_row[vec2d_index[1]] = 1;
    // } else if (num === 6) {
    //   new_frame_row[vec2d_index[1]] = 1;
    // } else {
    //   new_frame_row[vec2d_index[1]] = 0;
    // }
    if (num >= 2.68 && num < 3.4 && vec[vec2d_index[0]][vec2d_index[1]] > 0) {
      new_frame_row[vec2d_index[1]] = 1;
    } else if (num >= 3.4 && num <= 3.401) {
      new_frame_row[vec2d_index[1]] = 1;
    } else {
      new_frame_row[vec2d_index[1]] = 0;
    }
  };

  vec.forEach((vec2d_row, ri) => {
    const new_frame_row = new_frame[ri];

    const rows_for_access: (undefined | number[])[] = [];
    for (
      let index = ri - self_in_dm_index[1];
      index < ri - self_in_dm_index[1] + detection_matrix.length;
      index++
    ) {
      const element = vec[index];
      rows_for_access.push(element);
    }

    vec2d_row.forEach((_, ci) => {
      const vec2d_index = [ri, ci] as const;

      const matrix_data = get_matrix_data(
        detection_matrix,
        self_in_dm_index,
        shape,
        vec2d_index,
        rows_for_access
      );

      ruler(matrix_data, new_frame, new_frame_row, shape, vec2d_index);
    });
  });

  // debugger

  cells.vec = new_frame;
}

function next_frame() {
  function draw() {
    next_frame();
    // console.log("calc", after - before);
  }
  requestAnimationFrame(draw);
}

// clac_next_frame(cells.vec);

const canvas_container = ref<HTMLCanvasElement>();

onMounted(() => {
  const p5i = new p5((sk: p5) => {
    sk.setup = () => {
      sk.resizeCanvas(shape[0] * cell_size, shape[1] * cell_size);
      sk.noStroke();
      sk.colorMode(sk.HSB, 360, 1, 1, 1);
      // sk.fill(i++ % 365, 0.5, 0.8, 1);
      sk.fill(0, 0, 0, 1);
    };
    sk.draw = () => {
      sk.clear(0, 0, 0, 1);

      const before = performance.now();
      calc_next_frame(cells.vec);
      const after = performance.now();
      console.log("calc", after - before);

      let i = 0;

      cells.vec.forEach((row, ri) => {
        row.forEach((cell, ci) => {
          if (cell > 0) {
            sk.rect(ci * cell_size, ri * cell_size, cell_size, cell_size);
          }
        });
      });
    };
  }, canvas_container.value!);
});
next_frame();

// test();
</script>

<template>
  <div class="gol-page">
    <div></div>
    <div ref="canvas_container"></div>
  </div>
</template>

<style>
.gol-page {
  
}
</style>
